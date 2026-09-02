# Maintainer notes

Development and provenance notes for `dREG-apptainer`. Users don't need any of this
— see [`../README.md`](../README.md).

## Contents

- [Building the image](#building-the-image)
- [Why this wraps a prebuilt Docker image](#why-this-wraps-a-prebuilt-docker-image)
- [The image has no complete build recipe](#the-image-has-no-complete-build-recipe)
- [Archival](#archival)
- [GPU diagnostic record](#gpu-diagnostic-record)

## Building the image

The canonical form is an unprivileged build that skips the container-internal
`fakeroot` helper:

```bash
apptainer build --fakeroot --ignore-fakeroot-command dreg.sif dreg.def
```

`--ignore-fakeroot-command` avoids a failure where Apptainer injects a host `faked`
binary needing newer glibc than Ubuntu 20.04 provides. If the build dies in `%post`
with `/.singularity.d/libs/faked` and `GLIBC_2.33`/`GLIBC_2.34 not found`, that flag
is the fix — the error comes from Apptainer, not dREG.

A privileged build works, as does SingularityCE:

```bash
sudo apptainer build dreg.sif dreg.def
singularity build dreg.sif dreg.def
```

Requires a Linux x86_64 host: the base image is `amd64`, and Apptainer needs Linux
namespaces so there is no macOS path. Budget ~10 GB of scratch for the build plus
the multi-GB output. On HPC, point `APPTAINER_CACHEDIR` and `APPTAINER_TMPDIR` at
scratch if `$HOME` is quota-limited — the layer cache alone is 3.7 GB.

### Verifying a build

```bash
apptainer test dreg.sif                                      # loadability only
apptainer run dreg.sif --help
apptainer exec --nvccli --cleanenv dreg.sif dreg gpu_check   # the real check
```

`%test` checks **loadability only**. `requireNamespace()` never creates a CUDA
context, so `apptainer test` passes on a build whose GPU path is entirely dead —
this actually happened. `dreg gpu_check` trains a tiny Rgtsvm SVM and must run on a
GPU host; nvcc compiles fine without a GPU, but nothing validates the GPU without
one. Extend `%test` when adding a dependency rather than adding a separate harness.

## Why this wraps a prebuilt Docker image

dREG's R/CUDA stack is old and fragile, as the upstream tracker records:
[issue #19](https://github.com/Danko-Lab/dREG/issues/19) asks for known-working
versions because current R packages may not work;
[PR #18](https://github.com/Danko-Lab/dREG/pull/18) redirects `rphast` to
`CshlSiepelLab/RPHAST` after CRAN pulled it; and
[issue #20](https://github.com/Danko-Lab/dREG/issues/20) confirms the non-R tools
(`sort-bed` from BEDOPS, `tabix` from htslib, `bedGraphToBigWig` from UCSC).

BioHPC built their image with Ubuntu 20.04, R 4.0.5, CUDA 11.1, the then-current
Rgtsvm, and dREG 20200515. Docker Hub carries one `latest` tag, pushed 2022-11-05,
and the recipe pins its digest
(`sha256:ebcc18ebb774fa198d36f7398eb92f50b33fb368aab1d67efebedb7aceb5ba16`) so the
fragile package set never has to be reassembled from today's CRAN.

Do not switch to `:latest`, and do not add `%post` steps that install or upgrade R
packages — that defeats the reason this wraps a prebuilt image. If the digest is
bumped, update the matching `%labels` (`DockerDigest`, `BioHPCVersion`,
`BioHPCBase`) and every digest reference in the docs together.

## The image has no complete build recipe

This is stronger than "hard to rebuild." The image is 9 layers, 3.67 GB compressed.
Layers 1–8 are ordinary
[rocker-versioned2](https://github.com/rocker-org/rocker-versioned2) steps with
public scripts: Ubuntu 20.04, `install_R_source.sh` for R 4.0.5 (from a Posit
Package Manager snapshot dated 2021-05-17), then `install_cuda-11.1.sh` for the
2.7 GB CUDA layer.

Layer 9 is different. It is 633.6 MB, dated 2022-11-05 — five days after the others
— and its `created_by` field is just `R`, the inherited `CMD` rather than any build
instruction. That is the signature of `docker commit` on a running container. dREG,
Rgtsvm, rphast, bigWig, BEDOPS, htslib and the UCSC tools all live in that layer,
and **no build command for it was ever recorded.**

The layer has since been downloaded and inventoried, with the full manifest in
[`reference-stack-manifest.md`](reference-stack-manifest.md). Both git checkouts
survive inside the image, which pins the two hardest components to public commits:

| Component | Version | Source |
| --- | --- | --- |
| dREG | 1.4.0 | [`Danko-Lab/dREG@ab6dc2f3`](https://github.com/Danko-Lab/dREG/commit/ab6dc2f383772deb67f0c445c80e650cc054e762) (2021-10-11) |
| Rgtsvm | 0.55 | [`Danko-Lab/Rgtsvm@9cfaa95f`](https://github.com/Danko-Lab/Rgtsvm/commit/9cfaa95fa9859b2059fb71ece3829a775e28b85d) (2021-08-04) |
| rphast | 1.6.9 | ex-CRAN; `CshlSiepelLab/RPHAST` |
| bigWig | 0.2-9 | `Danko-Lab/BigWig-R-package` |
| htslib | 1.16 | provides `tabix`, `bgzip` |

Both commits were still reachable on GitHub as of 2026-08-31, and all 84 installed R
package versions are in the manifest. Note that `%labels BioHPCVersion 20200515` is
BioHPC's version string, **not** a dREG source date — the checkout is from
2021-10-11 and the image was built 2022-11-05.

Pinning the digest guards against the `latest` tag being *mutated*, not against it
being *deleted*. If Docker Hub drops the digest, the manifest means every
component's source and version is still known, so a rebuild is a describable task
rather than guesswork. What would be lost is the *built* artifact — above all Rgtsvm
0.55 compiled against CUDA 11.1, which needs the CUDA 11.1 toolchain and a driver
old enough to pair with it. The binary is the expensive thing, not the recipe, and
that is what archival insures against.

<details>
<summary>Verify the build history against the live registry</summary>

```bash
TOK=$(curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:biohpc/dreg:pull" | jq -r .token)
DIGEST=sha256:ebcc18ebb774fa198d36f7398eb92f50b33fb368aab1d67efebedb7aceb5ba16
CFG=$(curl -s -H "Authorization: Bearer $TOK" \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  "https://registry-1.docker.io/v2/biohpc/dreg/manifests/$DIGEST" | jq -r .config.digest)
curl -s -L -H "Authorization: Bearer $TOK" \
  "https://registry-1.docker.io/v2/biohpc/dreg/blobs/$CFG" | jq -r '.history[].created_by'
```

The last line of output is the bare `R` that marks layer 9.

</details>

## Archival

There is no distribution channel beyond Zenodo and deliberately no CI. pydreg is
what people should install, so building a release pipeline for this container would
be effort spent in the wrong place; the only thing worth doing is ensuring the
reference stack cannot be lost. A script and two GitHub Actions workflows for this
were written and deliberately removed — don't re-add them.

The SIF is archived alongside the model it needs, in the Zenodo record that also
hosts `asvm.gdm.6.6M.20170828.rdata`. Current version, published 2026-09-01:
[10.5281/zenodo.22225970](https://doi.org/10.5281/zenodo.22225970) — `dreg.sif`
(3.6 GB, `md5:2cb1ca16fb4c880b96faedaa90a094bd`) plus the model (570 MB,
`md5:da81e96de4988021fc1e53d5f79b77ad`). The concept DOI
[10.5281/zenodo.10113378](https://doi.org/10.5281/zenodo.10113378) always resolves to
the newest version.

Build on a GPU host so the GPU path can be verified before archiving:

```bash
apptainer build --fakeroot --ignore-fakeroot-command dreg.sif dreg.def
apptainer exec --nvccli --cleanenv dreg.sif dreg gpu_check
md5sum dreg.sif      # Zenodo records md5; note it for the description
```

Zenodo records are immutable once published, so adding a file means publishing a new
version:

1. Open the record and click **New version**. The draft inherits the existing model
   file — leave it in place.
2. Upload the SIF into that draft with
   [`zenodo-upload`](https://github.com/jhpoelen/zenodo-upload), which streams large
   files and retries. Pass the **draft's** deposition ID, not the published record
   ID; a published record's bucket is read-only.

   ```bash
   ZENODO_TOKEN=... ./zenodo_upload.sh <draft deposition id> dreg.sif
   ```

   Rehearse against sandbox first with `ZENODO_ENDPOINT=https://sandbox.zenodo.org`.
   The script runs under `set -x` and passes the token as a URL parameter, so it
   will be echoed to your terminal.
3. In the Zenodo UI, update the version string, append the SIF's SHA-256 and the
   base image digest to the description, then publish. **Publishing is
   irreversible** — Zenodo records cannot be deleted and files cannot be changed
   afterwards.

The uploaded filename is taken from the path you pass, so keep it `dreg.sif` to match
the existing record and the README.

After publishing, update the version-specific record ID in the README's
[Get the image](../README.md#1-get-the-image-and-the-model) section and in the
paragraph above — the concept DOI does not need changing, but direct file URLs are
per-version.

## GPU diagnostic record

Investigated 2026-08-31 on `cbsugpu01`: driver 580.167.08 / CUDA 13.0, TITAN Xp +
TITAN X (Pascal, `sm_61`). Symptom under `--nv`:

```
terminate called after throwing an instance of 'GTSVM::CUDA::Exception'
  what():  svm.cpp:735: Failed to allocate space for found keys on host
           (CUDA driver version is insufficient for CUDA runtime version)
```

**Cause:** Apptainer's `--nv` bind-mounts a fixed driver library list from
`nvliblist.conf` which is incomplete for this driver. `--nvccli` uses
`nvidia-container-cli` and works; so does `docker run --gpus all` on the same image.
The fault is in `--nv`'s injection, not in the image, the driver, or Rgtsvm.

The error message is a red herring. The failing call is a fixed **16 KB**
`cudaMallocHost` in a constructor — the first CUDA call in the process — so it
reports whatever sticky initialisation error is pending.

Ruled out, with evidence, so nobody repeats this:

| Hypothesis | Verdict | Evidence |
| --- | --- | --- |
| Driver too old for CUDA 11.1 | No | `cudaDriverGetVersion`=13000 vs `cudaRuntimeGetVersion`=11010 |
| CUDA stub shadowing the driver | No | stub dir contains no `libcuda.so.1` at all; real 96 MB `libcuda.so.1` loads from `/.singularity.d/libs` |
| Host/conda `LD_LIBRARY_PATH` leakage | No | `LD_LIBRARY_PATH` empty; `--cleanenv` changes nothing |
| GPU architecture / PTX JIT | No | rebuilt Rgtsvm PTX-only `compute_60`, confirmed loaded, identical failure; C kernels JIT fine for `sm_50` and `sm_61` |
| cudart linkage (static vs shared) | No | both static; the working C test links identically |
| dREG's usage | No | bare `Rgtsvm::svm()` on 100 random points fails identically |
| Allocation size / memory pressure | No | fixed 16 KB `cudaMallocHost` |
| NVBLAS pre-initialising CUDA | No | R uses `/usr/lib/x86_64-linux-gnu/blas/libblas.so.3.9.0`; unsetting changed nothing |

A standalone CUDA C program in the same container under `--nv` completes context
creation, pinned allocation, kernel launch and PTX JIT without error. Only in-process
CUDA calls from R fail. That narrowing is what pointed at the injection mechanism.

Two dead ends worth recording because they look plausible:

- **The `sm_52` cubin in `Rgtsvm.so` is not a defect.** `cuobjdump --list-elf` shows
  6 cubins (one `sm_52`) against 5 `sm_60` PTX modules. The `sm_52` one is generated
  by the device-link step, which nvcc runs without `-arch`, so it gets nvcc 11.x's
  default. A rebuild reproduces it. The shipped binary needs no rebuild.
- **Rebuilding Rgtsvm doesn't help, but if you ever need to:** the package lives at
  `/Rgtsvm/Rgtsvm` (a subdirectory of the repo root), Boost 1.71 is already at
  `/usr` but `configure` defaults to `/usr/local/boost`, and the source tree ships
  prebuilt `.o` files. So:
  `R CMD INSTALL --preclean --configure-args="--with-cuda-arch=compute_60 --with-boost-home=/usr" /Rgtsvm/Rgtsvm`
