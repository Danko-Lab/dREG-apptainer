# Reference stack manifest — `biohpc/dreg` layer 9

Layer 9 of the pinned base image (`sha256:7879d648…`, 633.6 MB, 2022-11-05) is a
`docker commit` snapshot with no recorded build command. This file is the
manifest of what it contains, recovered by downloading the layer blob directly
from the registry and reading the installed package metadata.

See [`MAINTENANCE.md`](MAINTENANCE.md) for why this
matters. Recovered 2026-08-31.

## Core components

| Component | Version | Provenance |
| --- | --- | --- |
| dREG | 1.4.0 | [`Danko-Lab/dREG@ab6dc2f3`](https://github.com/Danko-Lab/dREG/commit/ab6dc2f383772deb67f0c445c80e650cc054e762) — committed 2021-10-11 (*"error occurs if too few reads"*) |
| Rgtsvm | 0.55 | [`Danko-Lab/Rgtsvm@9cfaa95f`](https://github.com/Danko-Lab/Rgtsvm/commit/9cfaa95fa9859b2059fb71ece3829a775e28b85d) — committed 2021-08-04 (*"multiple GPUs"*) |
| rphast | 1.6.9 | pulled from CRAN; source at `CshlSiepelLab/RPHAST` |
| bigWig | 0.2-9 | `Danko-Lab/BigWig-R-package` |
| htslib | 1.16 | provides `tabix`, `bgzip` (`/usr/local/bin`) |
| BEDOPS | (bundled) | provides `sort-bed`, `bedops` (`/usr/bin`) |
| UCSC tools | (bundled) | provides `bedGraphToBigWig` (`/usr/local/bin`) |
| R | 4.0.5 | layer 3, rocker-versioned2; CRAN snapshot 2021-05-17 |
| CUDA | 11.1 | layer 6, rocker-versioned2 |
| Ubuntu | 20.04 | layer 1 |

Both git checkouts survive inside the image at `/dREG/.git` and `/Rgtsvm/.git`,
which is how the commit SHAs above were recovered. Note that Rgtsvm is built from
`master`, not from the `v0.4` tag that also appears in its refs.

### Version-label discrepancy

`%labels BioHPCVersion 20200515` in `dreg.def` is BioHPC's own version string and
does **not** describe the dREG source: the installed checkout is from 2021-10-11,
and the image was built 2022-11-05. Do not read `20200515` as a source date.

## Runtime note: use `--nvccli`

The device code in the shipped `Rgtsvm.so` is sound and needs no rebuild. Its
module set is 6 cubins (one `sm_52`, from nvcc's un-`-arch`'d device-link step)
and 5 `sm_60` PTX modules, which covers Pascal onward via cubin minor-version
compatibility and PTX JIT.

The failure mode seen on modern drivers is an Apptainer `--nv` problem, not a
defect in this stack: `--nv` bind-mounts an incomplete driver library set and
Rgtsvm aborts at its first CUDA call. Use `--nvccli`. See the troubleshooting
record in [`MAINTENANCE.md`](MAINTENANCE.md) for the full elimination table.

## Full R site-library contents

84 packages under `/usr/local/lib/R/site-library`:

| Package | Version | Package | Version |
| --- | --- | --- | --- |
| `askpass` | 1.1 | `pillar` | 1.6.1 |
| `bigWig` | 0.2-9 | `pkgbuild` | 1.2.0 |
| `bit` | 4.0.4 | `pkgconfig` | 2.0.3 |
| `bit64` | 4.0.5 | `pkgload` | 1.2.1 |
| `brew` | 1.0-6 | `praise` | 1.0.0 |
| `brio` | 1.1.2 | `prettyunits` | 1.1.1 |
| `cachem` | 1.0.5 | `processx` | 3.5.2 |
| `callr` | 3.7.0 | `proxy` | 0.4-25 |
| `cli` | 2.5.0 | `ps` | 1.6.0 |
| `clipr` | 0.7.1 | `purrr` | 0.3.4 |
| `commonmark` | 1.7 | `R6` | 2.5.0 |
| `crayon` | 1.4.1 | `randomForest` | 4.6-14 |
| `credentials` | 1.3.0 | `rcmdcheck` | 1.3.3 |
| `curl` | 4.3.1 | `rematch2` | 2.1.2 |
| `data.table` | 1.14.0 | `remotes` | 2.3.0 |
| `desc` | 1.3.0 | `Rgtsvm` | 0.55 |
| `devtools` | 2.4.1 | `rlang` | 0.4.11 |
| `diffobj` | 0.3.4 | `rmutil` | 1.1.5 |
| `digest` | 0.6.27 | `roxygen2` | 7.1.1 |
| `dREG` | 1.4.0 | `rphast` | 1.6.9 |
| `e1071` | 1.7-6 | `rprojroot` | 2.0.2 |
| `ellipsis` | 0.3.2 | `rstudioapi` | 0.13 |
| `evaluate` | 0.14 | `rversions` | 2.0.2 |
| `fansi` | 0.4.2 | `sessioninfo` | 1.1.1 |
| `fastmap` | 1.1.0 | `snow` | 0.4-3 |
| `fs` | 1.5.0 | `snowfall` | 1.84-6.1 |
| `gert` | 1.3.0 | `SparseM` | 1.81 |
| `gh` | 1.3.0 | `stringi` | 1.6.1 |
| `gitcreds` | 0.1.1 | `stringr` | 1.4.0 |
| `glue` | 1.4.2 | `sys` | 3.4 |
| `highr` | 0.9 | `testthat` | 3.0.2 |
| `httr` | 1.4.2 | `tibble` | 3.1.2 |
| `ini` | 0.3.1 | `usethis` | 2.0.1 |
| `knitr` | 1.33 | `utf8` | 1.2.1 |
| `lifecycle` | 1.0.0 | `vctrs` | 0.3.8 |
| `magrittr` | 2.0.1 | `waldo` | 0.2.5 |
| `markdown` | 1.1 | `whisker` | 0.4 |
| `Matrix` | 1.3-3 | `xfun` | 0.23 |
| `memoise` | 2.0.0 | `xml2` | 1.3.2 |
| `mime` | 0.10 | `xopen` | 1.0.0 |
| `mvtnorm` | 1.1-1 | `yaml` | 2.2.1 |
| `openssl` | 1.4.4 | `zip` | 2.1.1 |
