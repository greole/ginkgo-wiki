![Ginkgo](https://github.com/ginkgo-project/ginkgo/raw/develop/assets/logo.png)
===============================================================================

The Ginkgo team is proud to announce the first release of Ginkgo, the next-generation high-performance on-node sparse linear algebra library. Ginkgo leverages the features of modern C++ to give you a tool for the iterative solution of linear systems that is:

* __Easy to use.__ Interfaces with cryptic naming schemes and dozens of parameters are a thing of the past. Ginkgo was build with good software design in mind, making simple things simple to express.
* __High performance.__ Our optimized CUDA kernels ensure you are reaching the potential of today's GPU-accelerated high-end systems, while its design leaves it open to extension to any future architectures.
* __Controllable.__  While Ginkgo can automatically moves your data when needed, you remain in control by optionally specifying when the data is moved and what is its ownership scheme.
* __Composable.__ Iterative solution of linear systems is an extremely versatile field, where effective methods are built by mixing and match various components. Need a GMRES solver preconditioned with a block-Jacobi enhanced BiCGSTAB? Thanks to its novel linear operator abstraction, Ginkgo can do it!
* __Extensible.__ Did not find a component you were looking for? Ginkgo is designed to be easily extended in various ways. You can provide your own loggers, stopping criteria, matrix formats, preconditioners and solvers to Ginkgo and have them integrate as well as the natively supported ones, without the need to modify or recompile the library.

Ease of Use
-----------

Ginkgo uses high level abstractions to develop an efficient and understandable vocabulary for high-performance iterative solution of linear systems. As a result, the solution of a system stored in [matrix market format](https://math.nist.gov/MatrixMarket/formats.html) via a preconditioned Krylov solver on an accelerator is only [20 lines of code away](https://github.com/ginkgo-project/ginkgo/blob/develop/examples/minimal_solver_cuda/minimal_solver_cuda.cpp):

```c++
#include <ginkgo/ginkgo.hpp>
#include <iostream>

int main()
{
    // Instantiate a CUDA executor
    auto gpu = gko::CudaExecutor::create(0, gko::OmpExecutor::create());
    // Read data
    auto A = gko::read<gko::matrix::Csr<>>(std::cin, gpu);
    auto b = gko::read<gko::matrix::Dense<>>(std::cin, gpu);
    auto x = gko::read<gko::matrix::Dense<>>(std::cin, gpu);
    // Create the solver
    auto solver =
        gko::solver::Cg<>::build()
            .with_preconditioner(gko::preconditioner::Jacobi<>::build().on(gpu))
            .with_criteria(
                gko::stop::Iteration::build().with_max_iters(1000u).on(gpu),
                gko::stop::ResidualNormReduction<>::build()
                    .with_reduction_factor(1e-15)
                    .on(gpu))
            .on(gpu);
    // Solve system
    solver->generate(give(A))->apply(lend(b), lend(x));
    // Write result
    write(std::cout, lend(x));
}
```

Notice that Ginkgo is not a tool that generates C++. It _is_ C++. So just [install the library](https://github.com/ginkgo-project/ginkgo/blob/develop/README.md) (which is extremely simple due to its CMake-based build system), include the header and start using Ginkgo in your projects.

Already have an existing application and want to use Ginkgo to implement some part of it? Check out our [integration example](https://github.com/ginkgo-project/ginkgo/blob/develop/examples/3pt_stencil/3pt_stencil.cpp#L177) for a demonstration on how Ginkgo can be used with raw data already available in the application. If your data is in one of the formats supported by Ginkgo, it may be possible to use it directly, without creating a Ginkgo-dedicated copy of it.

