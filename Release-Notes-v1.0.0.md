![Ginkgo](https://github.com/ginkgo-project/ginkgo/raw/develop/assets/logo.png)
===============================================================================

The Ginkgo team is proud to announce the first release of Ginkgo, the next-generation high-performance on-node sparse linear algebra library. Ginkgo leverages the features of modern C++ to give you a tool for the iterative solution of linear systems that is:

* __Easy to use.__ Interfaces with cryptic naming schemes and dozens of parameters are a thing of the past. Ginkgo was build with good software design in mind, making simple things simple to express.
* __High performance.__ Our optimized CUDA kernels ensure you are reaching the potential of today's GPU-accelerated high-end systems, while Ginkgo's open design allows extension to future hardware architectures.
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

Designed for HPC
----------------

Ginkgo is designed to quickly adapt to rapid changes in the HPC architecture. Every component in Ginkgo is build around the _executor_ abstraction which is used to describe the execution and memory spaces where the operations are run, and the programming model used to realize the operations. The low-level performance critical kernels are implemented directly using each executor's programming model, while the high-level operation use a unified implementation that calls the low-level kernels. Consequently, the cost of developing new algorithms and extending existing ones to new architectures is kept relatively low, without compromising performance.
Currently, Ginkgo supports CUDA, reference and OpenMP executors.

The CUDA executor features highly-optimized kernels able to efficiently utilize NVIDIA's latest hardware. Several of these kernels appeared in recent scientific publications, including the optimized COO and CSR SpMV, and the block-Jacobi preconditioner with its adaptive precision version.

The reference executor can be used to verify the correctness of the code. It features a straightforward single threaded C++ implementation of the kernels which is easy to understand. As such, it can be used as a baseline for implementing other executors, verifying their correctness, or figuring out if unexpected behavior is the result of a faulty kernel or an error in the user's code.

Ginkgo 1.0.0 also offers initial support for the OpenMP executor. OpenMP kernels are currently implemented as minor modifications of the reference kernels with OpenMP pragmas and are considered experimental. Full OpenMP support with highly-optimized kernels is reserved for a future release.

Memory Management
-----------------

As a result of its executor-based design and high level abstractions, Ginkgo has explicit information about the location of every piece of data it needs and can automatically allocate, free and move the data where it is needed. However, lazily moving data around is often not optimal, and determining when a piece of data should be copied or shared in general cannot be done automatically. For this reason, Ginkgo also gives explicit control of sharing and moving its objects to the user via the dedicated ownership commands: `gko::clone`, `gko::share`, `gko::give` and `gko::lend`. If you are interested in a detailed description of the problems the C++ standard has with these concepts check out [this Ginkgo Wiki page](https://github.com/ginkgo-project/ginkgo/wiki/Library-design#use-of-pointers), and for more details about Ginkgo's solution to the problem and the description of ownership commands take a look at [this issue](https://github.com/ginkgo-project/ginkgo/issues/30).

