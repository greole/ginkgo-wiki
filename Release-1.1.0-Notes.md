The Ginkgo team is proud to announce the new minor release of Ginkgo version
1.1.0. This release brings several performance improvements, adds Windows support, 
adds support for factorizations inside Ginkgo and a new ILU preconditioner
based on ParILU algorithm, among other things. For detailed information, check the respective issue.

Supported systems and requirements:
+ For all platforms, cmake 3.9+
+ Linux and MacOS
  + gcc: 5.3+, 6.3+, 7.3+, 8.1+
  + clang: 3.9+
  + Intel compiler: 2017+
  + Apple LLVM: 8.0+
  + CUDA module: CUDA 9.0+
+ Windows
  + MinGW and CygWin: gcc 5.3+, 6.3+, 7.3+, 8.1+
  + Microsoft Visual Studio: VS 2017 15.7+
  + CUDA module: CUDA 9.0+, Microsoft Visual Studio
  + OpenMP module: MinGW or CygWin.


The current known issues can be found in the [known issues
page](https://github.com/ginkgo-project/ginkgo/wiki/Known-Issues).


Additions:
+ Upper and lower triangular solvers ([#327](https://github.com/ginkgo-project/ginkgo/issues/327), [#336](https://github.com/ginkgo-project/ginkgo/issues/336), [#341](https://github.com/ginkgo-project/ginkgo/issues/341), [#342](https://github.com/ginkgo-project/ginkgo/issues/342)) 
+ New factorization support in Ginkgo, and addition of the ParILU
  algorithm ([#305](https://github.com/ginkgo-project/ginkgo/issues/305), [#315](https://github.com/ginkgo-project/ginkgo/issues/315), [#319](https://github.com/ginkgo-project/ginkgo/issues/319), [#324](https://github.com/ginkgo-project/ginkgo/issues/324))
+ New ILU preconditioner ([#348](https://github.com/ginkgo-project/ginkgo/issues/348), [#353](https://github.com/ginkgo-project/ginkgo/issues/353))
+ Windows MinGW and Cygwin support ([#347](https://github.com/ginkgo-project/ginkgo/issues/347))
+ Windows Visual studio support ([#351](https://github.com/ginkgo-project/ginkgo/issues/351))
+ New example showing how to use ParILU as a preconditioner ([#358](https://github.com/ginkgo-project/ginkgo/issues/358))
+ New example on using loggers for debugging ([#360](https://github.com/ginkgo-project/ginkgo/issues/360))
+ Add two new 9pt and 27pt stencil examples ([#300](https://github.com/ginkgo-project/ginkgo/issues/300), [#306](https://github.com/ginkgo-project/ginkgo/issues/306))
+ Allow benchmarking CuSPARSE spmv formats through Ginkgo's benchmarks ([#303](https://github.com/ginkgo-project/ginkgo/issues/303))
+ New benchmark for sparse matrix format conversions ([#312](https://github.com/ginkgo-project/ginkgo/issues/312)[#317](https://github.com/ginkgo-project/ginkgo/issues/317))
+ Add conversions between CSR and Hybrid formats ([#302](https://github.com/ginkgo-project/ginkgo/issues/302), [#310](https://github.com/ginkgo-project/ginkgo/issues/310))
+ Support for sorting rows in the CSR format by column idices ([#322](https://github.com/ginkgo-project/ginkgo/issues/322))
+ Addition of a CUDA COO SpMM kernel for improved performance ([#345](https://github.com/ginkgo-project/ginkgo/issues/345))
+ Addition of a LinOp to handle perturbations of the form (identity + scalar *
  basis * projector) ([#334](https://github.com/ginkgo-project/ginkgo/issues/334))
+ New sparsity matrix representation format with Reference and OpenMP
  kernels ([#349](https://github.com/ginkgo-project/ginkgo/issues/349), [#350](https://github.com/ginkgo-project/ginkgo/issues/350))

Fixes:
+ Accelerate GMRES solver for CUDA executor ([#363](https://github.com/ginkgo-project/ginkgo/issues/363))
+ Fix BiCGSTAB solver convergence ([#359](https://github.com/ginkgo-project/ginkgo/issues/359))
+ Fix CGS logging by reporting the residual for every sub iteration ([#328](https://github.com/ginkgo-project/ginkgo/issues/328))
+ Fix CSR,Dense->Sellp conversion's memory access violation ([#295](https://github.com/ginkgo-project/ginkgo/issues/295))
+ Accelerate CSR->Ell,Hybrid conversions on CUDA ([#313](https://github.com/ginkgo-project/ginkgo/issues/313), [#318](https://github.com/ginkgo-project/ginkgo/issues/318))
+ Fixed slowdown of COO SpMV on OpenMP ([#340](https://github.com/ginkgo-project/ginkgo/issues/340))
+ Fix gcc 6.4.0 internal compiler error ([#316](https://github.com/ginkgo-project/ginkgo/issues/316))
+ Fix compilation issue on Apple clang++ 10 ([#322](https://github.com/ginkgo-project/ginkgo/issues/322))
+ Make Ginkgo able to compile on Intel 2017 and above ([#337](https://github.com/ginkgo-project/ginkgo/issues/337))
+ Make the benchmarks spmv/solver use the same matrix formats ([#366](https://github.com/ginkgo-project/ginkgo/issues/366))
+ Fix self-written isfinite function ([#348](https://github.com/ginkgo-project/ginkgo/issues/348))

Tools and ecosystem:
+ Multiple improvements to the CI system and tools ([#296](https://github.com/ginkgo-project/ginkgo/issues/296), [#311](https://github.com/ginkgo-project/ginkgo/issues/311), [#365](https://github.com/ginkgo-project/ginkgo/issues/365))
+ Multiple improvements to the Ginkgo containers ([#328](https://github.com/ginkgo-project/ginkgo/issues/328), [#361](https://github.com/ginkgo-project/ginkgo/issues/361))
+ Add sonarqube analysis to Ginkgo ([#304](https://github.com/ginkgo-project/ginkgo/issues/304), [#308](https://github.com/ginkgo-project/ginkgo/issues/308), [#309](https://github.com/ginkgo-project/ginkgo/issues/309))
+ Add clang-tidy and iwyu support to Ginkgo ([#298](https://github.com/ginkgo-project/ginkgo/issues/298))
+ Improve Ginkgo's support of xSDK M12 policy by adding the `TPL_` arguments
  to CMake ([#300](https://github.com/ginkgo-project/ginkgo/issues/300))
+ Add support for the xSDK R7 policy ([#325](https://github.com/ginkgo-project/ginkgo/issues/325))
+ Fix examples in html documentation ([#367](https://github.com/ginkgo-project/ginkgo/issues/367))