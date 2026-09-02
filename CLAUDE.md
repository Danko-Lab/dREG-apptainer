# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A single Apptainer/Singularity definition file (`dreg.def`) that wraps the Cornell BioHPC Docker image `biohpc/dreg` for [Danko-Lab/dREG](https://github.com/Danko-Lab/dREG). There is no application source code here — the deliverable is the recipe plus the `README.md` that documents build/run/troubleshooting workflows.

**This repo is superseded and is not the recommended way to run dREG.** [pydreg](https://github.com/adamyhe/pydreg) (`pip install pydreg` / `uv tool install pydreg`, same author) is a Python reimplementation that installs in seconds, runs substantially faster, and agrees with this container's R implementation to >0.999 Jaccard on peak calls across 12 real libraries at default parameters. Point users there.

Consequently, do **not** build distribution machinery for this image — no CI, no GHCR/ORAS push, no registry tags, no signing-for-users. That work was started and deliberately deleted (2026-08-31): shipping a 4 GB CUDA 11.1 container to users who could `pip install pydreg` is a disservice, and the effort is a trap that looks productive. The repo's remaining purpose is narrow and archival:

1. Reproducing pre-pydreg published results, within stochastic tolerance -- *not* bit-for-bit. dREG and pydreg both use unseeded RNG, so successive runs of either differ slightly; exact reproduction is impossible by any route. Don't claim otherwise in docs.
2. Settling the non-default parameter regions pydreg's own README flags as unvalidated against R — only this container can.
3. Preserving the stack (see the layer-9 bullet below).

Archival is a hand-run, occasional task, not a pipeline. The SIF is published in the Zenodo record that also hosts the dREG model, owned by this repo's author: concept DOI [10.5281/zenodo.10113378](https://doi.org/10.5281/zenodo.10113378), current version [10.5281/zenodo.22225970](https://doi.org/10.5281/zenodo.22225970) (`dreg.sif`, 3.6 GB, published 2026-09-01). New versions go up via the Zenodo UI's *New version* button plus [`zenodo-upload`](https://github.com/jhpoelen/zenodo-upload) for the large file; see `docs/MAINTENANCE.md`. Do not re-add a script or CI for this -- both were written and deliberately removed. Direct file URLs are version-specific, so bumping the SIF means updating the record ID in `README.md` and `docs/MAINTENANCE.md`; the concept DOI never changes.

## Commands

Build (unprivileged; this is the canonical form):

```bash
apptainer build --fakeroot --ignore-fakeroot-command dreg.sif dreg.def
```

Test / smoke check:

```bash
apptainer test dreg.sif        # %test: loadability only, NOT the GPU
apptainer run dreg.sif --help  # runscript -> dreg wrapper help
apptainer exec --nvccli --cleanenv dreg.sif dreg gpu_check   # real GPU test
apptainer exec --nvccli --cleanenv dreg.sif dreg cuda_check
```

Typical GPU peak-calling run (the correct invocation *for this container*; not the recommended way to call dREG peaks — see pydreg above):

```bash
apptainer exec --nvccli --cleanenv --bind /path/to/data:/data dreg.sif \
  dreg run_dREG /data/plus.bw /data/minus.bw /data/out /data/asvm.gdm.6.6M.20170828.rdata 16 0
```

The last two `run_dREG` args are NTHREADS and GPU ID; pick an idle GPU based on `nvidia-smi`.

There is no lint or unit-test suite. `%test` in `dreg.def` is the only automated verification: it asserts the three upstream `.bsh` scripts exist, that `Rscript`/`sort-bed`/`tabix`/`bedGraphToBigWig` are on PATH, and that the pinned R package set (`dREG`, `Rgtsvm`, `rphast`, `bigWig`, …) loads. Extend that block when adding a dependency rather than adding a separate test harness.

## Architecture and design constraints

- **Base image is digest-pinned on purpose.** `Bootstrap: docker` / `From: biohpc/dreg@sha256:ebcc18e...` freezes Ubuntu 20.04 + R 4.0.5 + CUDA 11.1 + dREG 20200515. dREG's R/CUDA stack cannot be rebuilt from today's CRAN state (rphast was pulled from CRAN; Rgtsvm needs an old CUDA). Do not switch to `:latest`, and do not add `%post` steps that install or upgrade R packages — that defeats the reason this wraps a prebuilt image. If the digest is ever bumped, update the matching `%labels` (`DockerDigest`, `BioHPCVersion`, `BioHPCBase`) and the README's digest reference together.
- **The pinned image's final layer is an unrecipe'd `docker commit`; its contents are inventoried in `docs/reference-stack-manifest.md`.** Layers 1-8 are public `rocker-versioned2` steps (Ubuntu 20.04, R 4.0.5 from a 2021-05-17 CRAN snapshot, CUDA 11.1). Layer 9 (633.6 MB, 2022-11-05) has `created_by: "R"` — the inherited `CMD`, not a build instruction — so dREG, Rgtsvm, rphast, bigWig, BEDOPS, htslib and the UCSC tools were installed interactively and committed with no recorded command. The layer was downloaded and inventoried 2026-08-31: `/dREG/.git` and `/Rgtsvm/.git` survive inside it, pinning dREG 1.4.0 to `Danko-Lab/dREG@ab6dc2f3` (2021-10-11) and Rgtsvm 0.55 to `Danko-Lab/Rgtsvm@9cfaa95f` (2021-08-04) — both confirmed reachable on GitHub — plus all 84 R package versions. So the *recipe* is now documented; what remains irreplaceable is the *built* Rgtsvm compiled against CUDA 11.1, which needs an old toolchain and a matching driver. Archive the SIF to preserve that build environment, not the information. Also: `%labels BioHPCVersion 20200515` is BioHPC's version string, not a dREG source date — don't propagate it as one.
- **`%post` writes a single artifact: `/usr/local/bin/dreg`.** This shell wrapper is the container's whole interface, and `%runscript` just `exec dreg "$@"`. It dispatches `run_dREG` / `run_predict` / `writeBed` to the corresponding `/dREG/*.bsh` scripts, provides `cuda_check` and `shell`, and falls through to `exec "$@"` for anything else. Because it is a heredoc inside `%post`, edits to it must keep the quoting intact (`<<'EOF'` — no host-side variable expansion).
- **Host R leakage is a known failure mode.** `R_LIBS_USER`, `R_PROFILE_USER`, and `R_ENVIRON_USER` are set in *both* `%environment` and the `dreg` wrapper so that a user's `~/R/x86_64-pc-linux-gnu-library/4.0` and `~/.Rprofile` cannot override the pinned stack. Keep both copies in sync; the wrapper's copy is what protects `apptainer exec` invocations that bypass `%environment`.
- **Run with `--nvccli`, never `--nv`.** With `--nv` on a modern driver (confirmed broken on 580.167.08 / CUDA 13.0, Pascal `sm_61`), Rgtsvm aborts at its first CUDA call with a misleading `CUDA driver version is insufficient for CUDA runtime version` at `svm.cpp:735`. The driver is fine: it reports CUDA 13.0 against the 11.1 runtime, and a standalone CUDA C program in the same container under `--nv` does context creation, pinned allocation, kernel launch and PTX JIT without error. Only in-process CUDA calls from R fail. `--nv` bind-mounts a fixed library list from Apptainer's `nvliblist.conf` which is incomplete for this driver; `--nvccli` uses `nvidia-container-cli` (as Docker's NVIDIA toolkit does) and works, as does `docker run --gpus all` on the same image. Ruled out and not worth re-testing: CUDA stubs (the stub dir has no `libcuda.so.1`), host/conda `LD_LIBRARY_PATH` leakage, GPU arch (a PTX-only `compute_60` rebuild of Rgtsvm changes nothing -- the shipped binary is fine and needs no rebuild), cudart linkage, NVBLAS, allocation size (fixed 16 KB). Investigated 2026-08-31.
- **`%test` cannot detect GPU faults.** It asserts packages import; `requireNamespace()` never creates a CUDA context, so it passes on a build whose GPU path is dead. `dreg gpu_check` (trains a tiny Rgtsvm SVM) is the real check and must run on a GPU host.
- **CUDA comes from the host driver only.** The wrapper prepends `/.singularity.d/libs` to `LD_LIBRARY_PATH` when Apptainer has bound driver libraries there, so the container's CUDA 11.1 runtime pairs with the host driver instead of CUDA stubs or a host CUDA module.
- **Nothing large is baked into the image.** The 2017 model file (`asvm.gdm.6.6M.20170828.rdata`) stays on the host and is bind-mounted; `*.sif` / `*.simg` are gitignored.

## Documentation expectations

Docs are split by audience, and that split is load-bearing:

- **`README.md` is for downstream users of the published image only** — get the SIF and model from Zenodo, check the GPU, call peaks, read the output files, troubleshoot. It keeps one short "or build it yourself" section and nothing more. Do not migrate provenance, diagnostics, or release procedure back into it.
- **`docs/MAINTENANCE.md`** holds all developer material: building, why the base image is wrapped, the layer-9 provenance, archival to Zenodo, and the GPU diagnostic record with its ruled-out hypotheses.
- **`docs/reference-stack-manifest.md`** is the inventory of layer 9 (versions, commits, all 84 R packages).

Behavioral changes to `dreg.def` should land with the corresponding doc update in the same commit — that is the pattern in the existing history. User-visible changes (flags, subcommands, outputs) go in `README.md`; everything else goes in `docs/`.
