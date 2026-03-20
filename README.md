# Tensile

> [!IMPORTANT]
> This repository is an experimental lab fork: `AETS-MAGI/Tensile-gfx900_aets-lab`.
>
> - Fork source: `ROCm/Tensile`
> - Purpose: gfx900/MI25 bring-up and ROCm runtime investigation
> - Branch policy: `main` as baseline, experiments on dedicated branches (for example `gfx900-bringup`)
> - License note: upstream `LICENSE` and copyright notices are preserved

> [!CAUTION]
> The Tensile repository is retired, please use the [ROCm/rocm-libraries](https://github.com/ROCm/rocm-libraries) repository

Tensile is a tool for creating benchmark-driven backend libraries for GEMMs, GEMM-like problems (such as batched GEMM), and general N-dimensional tensor contractions on a GPU.
The Tensile library is mainly used as a backend library for rocBLAS.
Tensile acts as the performance backbone for a wide variety of 'compute' applications running on AMD GPUs.

> [!NOTE]
> The published documentation is available at [Tensile](https://rocm.docs.amd.com/projects/Tensile/en/latest/index.html) in an organized, easy-to-read format, with search and a table of contents. The documentation source files reside in the `Tensile/docs/src` folder of this repository. As with all ROCm projects, the documentation is open source. For more information on contributing to the documentation, see [Contribute to ROCm documentation](https://rocm.docs.amd.com/en/latest/contribute/contributing.html).
