# dREG-apptainer

Apptainer recipe for [Danko-Lab/dREG](https://github.com/Danko-Lab/dREG), using the Cornell BioHPC Docker image [`biohpc/dreg`](https://hub.docker.com/r/biohpc/dreg) as the base.

## Why this wraps the Docker image

dREG has a very old R/CUDA dependency stack. The upstream issue tracker has a few important install notes:

- [Issue #19](https://github.com/Danko-Lab/dREG/issues/19) asks for known working versions because current R packages may not work with dREG.
- [PR #18](https://github.com/Danko-Lab/dREG/pull/18) updates `rDeps.R` to install `rphast` from `CshlSiepelLab/RPHAST`, since the CRAN package was pulled.
- [Issue #20](https://github.com/Danko-Lab/dREG/issues/20) confirms the non-R runtime tools: `sort-bed` from BEDOPS, `tabix` from htslib, and `bedGraphToBigWig` from UCSC tools.

BioHPC's dREG page says their Docker image was built with Ubuntu 20.04, R 4.0.5, CUDA 11.1, the latest Rgtsvm at build time, and dREG 20200515. Docker Hub currently has one `latest` tag, pushed 2022-11-05, and the recipe pins its digest: `sha256:ebcc18ebb774fa198d36f7398eb92f50b33fb368aab1d67efebedb7aceb5ba16`. Using that image avoids rebuilding the fragile R package set from today's CRAN/GitHub state.

## Build

For an unprivileged build, use Apptainer's fakeroot mode but skip the container-internal `fakeroot` helper:

```bash
apptainer build --fakeroot --ignore-fakeroot-command dreg.sif dreg.def
```

This avoids a known failure mode where Apptainer injects a host `faked` binary that requires newer glibc than the Ubuntu 20.04 base image provides.

If you have admin privileges on the build host, a privileged build is also fine:

```bash
sudo apptainer build dreg.sif dreg.def
```

If your cluster uses SingularityCE, the same definition should work:

```bash
singularity build dreg.sif dreg.def
```

### Fakeroot GLIBC Troubleshooting

If the build fails during `%post` with messages like `/.singularity.d/libs/faked` and `GLIBC_2.33` or `GLIBC_2.34 not found`, rebuild with:

```bash
apptainer build --fakeroot --ignore-fakeroot-command dreg.sif dreg.def
```

That error comes from Apptainer's fakeroot helper, not from dREG itself.

### Host R Library Troubleshooting

If R tries to load packages from your home directory, for example `/home/$USER/R/x86_64-pc-linux-gnu-library/4.0`, rebuild this image from the current `dreg.def`. The definition sets `R_LIBS_USER`, `R_PROFILE_USER`, and `R_ENVIRON_USER` so host R libraries and startup files do not override the container's pinned R stack.

For an already-built SIF, use this runtime workaround:

```bash
apptainer exec --nv --cleanenv \
  --env R_LIBS_USER=/tmp \
  --env R_PROFILE_USER=/dev/null \
  --env R_ENVIRON_USER=/dev/null \
  --bind /path/to/data:/data dreg.sif \
  dreg run_dREG /data/mysample_plus.bw /data/mysample_minus.bw /data/output_prefix /data/asvm.gdm.6.6M.20170828.rdata 16 0
```

## Test apptainer

Run the following commands to test the apptainer build.

```bash
apptainer test dreg.sif
apptainer run dreg.sif --help
```

On GPU-capable machines, also check CUDA visibility.

```bash
apptainer exec --nv dreg.sif nvidia-smi
```

If the host reports a new CUDA driver but dREG fails with `CUDA driver version is insufficient for CUDA runtime version`, check that the container is using Apptainer's bound driver libraries instead of CUDA stubs or host CUDA module libraries:

```bash
apptainer exec --nv --cleanenv dreg.sif \
  dreg cuda_check
```

Run dREG on an idle GPU. The final argument to `run_dREG` is the GPU ID; if `nvidia-smi` shows GPU 0 is busy and GPU 1 is idle, use `1`.

The `dreg` wrapper prepends `/.singularity.d/libs` to `LD_LIBRARY_PATH` when present. If the error persists, try Apptainer's NVIDIA container CLI integration if enabled on your cluster:

```bash
apptainer exec --nvccli --cleanenv --bind /path/to/data:/data dreg.sif \
  dreg run_dREG /data/mysample_plus.bw /data/mysample_minus.bw /data/output_prefix /data/asvm.gdm.6.6M.20170828.rdata 16 1
```

Avoid loading a host CUDA toolkit module before launching the container unless your cluster requires it; dREG should use the CUDA runtime in the image and only the NVIDIA driver from the host.

## Run

Bind your working directory into the container. For GPU peak calling, pass `--nv` so Apptainer exposes NVIDIA libraries/devices. (THIS IS THE RECOMMENDED CURRENT WORKFLOW).

```bash
apptainer exec --nv --bind /path/to/data:/data dreg.sif \
  dreg run_dREG /data/mysample_plus.bw /data/mysample_minus.bw /data/output_prefix /data/asvm.gdm.6.6M.20170828.rdata 16 0
```

For the legacy score-prediction workflow:

```bash
apptainer exec --bind /path/to/data:/data dreg.sif \
  dreg run_predict /data/mysample_plus.bw /data/mysample_minus.bw /data/output_prefix /data/asvm.RData 2
```

To call peaks from legacy bedGraph output:

```bash
apptainer exec --bind /path/to/data:/data dreg.sif \
  dreg writeBed 0.8 /data/output_prefix.bedGraph.gz
```

You can also run arbitrary commands inside the image:

```bash
apptainer exec dreg.sif Rscript -e 'library(dREG); sessionInfo()'
```

## Model File

The newer peak-calling workflow needs the 2017 dREG model. Upstream lists these sources:

- <https://dreg.dnasequence.org/themes/dreg/assets/file/asvm.gdm.6.6M.20170828.rdata>
- <ftp://cbsuftp.tc.cornell.edu/danko/hub/dreg.models/asvm.gdm.6.6M.20170828.rdata>
- <https://zenodo.org/records/10113379>

Keep the model outside the SIF and bind it in with your data; this keeps the image small and avoids baking large mutable research data into the container.
