# dREG-apptainer

A prebuilt Apptainer image of [dREG](https://github.com/Danko-Lab/dREG) — the
reference R/CUDA implementation, frozen with the exact R 4.0.5 / CUDA 11.1 stack it
was validated against. Download it, bind your data, call peaks.

> ### Most people should use pydreg instead
>
> ```bash
> uv tool install pydreg     # or: pip install pydreg
> ```
>
> [**pydreg**](https://github.com/adamyhe/pydreg) is a Python reimplementation that
> installs in seconds instead of needing a multi-GB CUDA container, runs
> substantially faster, works on Apple Silicon, and agrees with this image to
> **>0.999 Jaccard** on peak calls at default parameters.
>
> Use *this* image when you specifically need the R reference: reproducing published
> dREG 20200515 results, or validating pydreg against R. For calling peaks on your
> own data, use pydreg.

## 1. Get the image and the model

Both live in the same Zenodo record:
**[10.5281/zenodo.22225970](https://doi.org/10.5281/zenodo.22225970)**

```bash
apptainer pull dreg.sif \
  https://zenodo.org/records/22225970/files/dreg.sif
curl -LO https://zenodo.org/records/22225970/files/asvm.gdm.6.6M.20170828.rdata
```

That's 3.6 GB for the image and 570 MB for the model. Verify if you like:

```bash
md5sum dreg.sif asvm.gdm.6.6M.20170828.rdata
# dreg.sif                      2cb1ca16fb4c880b96faedaa90a094bd
# asvm.gdm.6.6M.20170828.rdata  da81e96de4988021fc1e53d5f79b77ad
```

The model is deliberately *not* baked into the image — keep it beside your data and
bind it in. It is also mirrored at
<https://dreg.dnasequence.org/themes/dreg/assets/file/asvm.gdm.6.6M.20170828.rdata>.

[10.5281/zenodo.10113378](https://doi.org/10.5281/zenodo.10113378) is the concept DOI
for the record and always resolves to the newest version; cite that one.

### Or build it yourself

On a Linux x86_64 host (Apptainer doesn't run on macOS, and the base image is
`amd64`):

```bash
apptainer build --fakeroot --ignore-fakeroot-command dreg.sif dreg.def
```

`--ignore-fakeroot-command` is not optional on most hosts: without it Apptainer
injects a host `faked` binary that needs newer glibc than Ubuntu 20.04 provides, and
`%post` fails with `GLIBC_2.33`/`GLIBC_2.34 not found`. `sudo apptainer build` and
`singularity build` also work. Expect a multi-GB image and ~10 GB of scratch during
the build. Details in [`docs/MAINTENANCE.md`](docs/MAINTENANCE.md).

## 2. Check the GPU works

Do this once, before a long run. It trains a tiny SVM through Rgtsvm — the first
thing in the stack that touches the GPU:

```bash
apptainer exec --nvccli --cleanenv dreg.sif dreg gpu_check
```

It should print an SVM summary and `OK: Rgtsvm created a CUDA context`. If it
doesn't, go to [Troubleshooting](#troubleshooting) — do not skip ahead, since
`apptainer test` passes even when the GPU path is completely dead.

## 3. Call peaks

```bash
apptainer exec --nvccli --cleanenv --bind /path/to/data:/data dreg.sif \
  dreg run_dREG /data/sample.pl.bw /data/sample.mn.bw /data/sample \
                /data/asvm.gdm.6.6M.20170828.rdata 16 0
```

| Argument | Meaning |
| --- | --- |
| `sample.pl.bw` | plus-strand PRO-seq/GRO-seq bigWig |
| `sample.mn.bw` | minus-strand bigWig |
| `sample` | output *prefix*, not a filename |
| `asvm...rdata` | the model file |
| `16` | CPU cores (optional, default 1) |
| `0` | GPU id (optional) |

Two things that catch people:

- **The GPU id is what enables GPU mode.** Omit it and dREG silently runs on CPU,
  which is orders of magnitude slower. Pass `0` to use the default GPU; pass a
  higher number to select a specific one if `nvidia-smi` shows GPU 0 is busy.
- **bigWigs must be raw read counts, not normalized**, with the minus strand
  negative. dREG validates this and aborts with *"bigWig files maybe not meet the
  requirements"*; see upstream
  [data preparation](https://github.com/Danko-Lab/dREG#data-preparation).

(The upstream usage text mentions a 7th `GPU.threads` argument. The script never
reads it — don't bother passing it.)

## 4. Output files

With prefix `sample`, you get these next to your data. Each `.bed.gz` is
bgzip-compressed and tabix-indexed (`.bed.gz.tbi` alongside):

| File | Contents |
| --- | --- |
| `sample.dREG.peak.full.bed.gz` | **the main result** — peaks with score and probability |
| `sample.dREG.peak.prob.bed.gz` | peaks with probability only (chrom, start, end, prob) |
| `sample.dREG.peak.score.bed.gz` | peaks with raw SVR score only |
| `sample.dREG.infp.bed.gz` | scores at every informative position |
| `sample.dREG.raw.peak.bed.gz` | raw peak calls before filtering |
| `sample.dREG.peak.prob.bw` | probability as a bigWig, for genome browsers |
| `sample.dREG.peak.score.bw` | score as a bigWig |
| `sample.dREG.infp.bw` | informative-position scores as a bigWig |
| `sample.chrom.info` | chromosome sizes derived from your bigWigs |

Most downstream work uses `*.dREG.peak.full.bed.gz` or `*.dREG.peak.prob.bed.gz`.

## Troubleshooting

### `CUDA driver version is insufficient` — use `--nvccli`, not `--nv`

If you launch with `--nv`, Rgtsvm aborts at its first CUDA call:

```
terminate called after throwing an instance of 'GTSVM::CUDA::Exception'
  what():  svm.cpp:735: Failed to allocate space for found keys on host
           (CUDA driver version is insufficient for CUDA runtime version)
```

The message is wrong — your driver is fine. `--nv` bind-mounts a fixed library list
from Apptainer's `nvliblist.conf`, which is incomplete for recent drivers.
`--nvccli` routes through `nvidia-container-cli` instead and works. **Always use
`--nvccli`.**

It implies `--writable-tmpfs` and sets `NVIDIA_VISIBLE_DEVICES=all`; both are
harmless. If your cluster has no `nvidia-container-cli`, `--nv` may still work on
older drivers — confirm with `dreg gpu_check` before committing to a long run.
`docker run --gpus all` on the same image also works.

Confirmed broken with `--nv` on driver 580.167.08 / CUDA 13.0 with Pascal GPUs. The
full diagnostic record, including what was ruled out, is in
[`docs/MAINTENANCE.md`](docs/MAINTENANCE.md).

### R loads packages from my home directory

If R picks up `~/R/x86_64-pc-linux-gnu-library/4.0` and breaks the pinned stack,
`--cleanenv` normally prevents it. To force the issue:

```bash
apptainer exec --nvccli --cleanenv \
  --env R_LIBS_USER=/tmp \
  --env R_PROFILE_USER=/dev/null \
  --env R_ENVIRON_USER=/dev/null \
  --bind /path/to/data:/data dreg.sif \
  dreg run_dREG /data/sample.pl.bw /data/sample.mn.bw /data/sample \
                /data/asvm.gdm.6.6M.20170828.rdata 16 0
```

### Other notes

- Don't load a host CUDA toolkit module before launching. The image supplies the
  CUDA runtime; only the driver should come from the host.
- Peak calling is memory-hungry on top of the GPU. If it dies partway, retry with
  fewer CPU cores before assuming a GPU problem.

## Other commands

Everything runs through one wrapper, `/usr/local/bin/dreg`:

| Command | Purpose |
| --- | --- |
| `dreg run_dREG PL.bw MN.bw PREFIX MODEL.rdata [CORES] [GPU]` | peak calling (above) |
| `dreg run_predict PL.bw MN.bw PREFIX MODEL.rdata [CORES] [GPU]` | legacy score prediction |
| `dreg writeBed THRESHOLD FILE.bedGraph.gz` | peaks from legacy bedGraph output |
| `dreg gpu_check` | prove the GPU works |
| `dreg cuda_check` | print `nvidia-smi`, `LD_LIBRARY_PATH`, CUDA library resolution |
| `dreg shell` | interactive bash in the container |
| anything else | run directly in the container |

Anything unrecognised is executed as-is, so the image doubles as a frozen R 4.0.5
environment:

```bash
apptainer exec dreg.sif Rscript -e 'library(dREG); sessionInfo()'
```

## What's inside

dREG 1.4.0, Rgtsvm 0.55, rphast 1.6.9, bigWig 0.2-9 on R 4.0.5 / CUDA 11.1 /
Ubuntu 20.04, from the Cornell BioHPC image
[`biohpc/dreg`](https://hub.docker.com/r/biohpc/dreg) pinned by digest. Exact
versions and commits for all 84 R packages are in
[`docs/reference-stack-manifest.md`](docs/reference-stack-manifest.md).

Building the image, why it wraps a prebuilt Docker image, and how it is archived are
covered in [`docs/MAINTENANCE.md`](docs/MAINTENANCE.md).
