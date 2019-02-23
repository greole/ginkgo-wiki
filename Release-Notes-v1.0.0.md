![Ginkgo](https://github.com/ginkgo-project/ginkgo/raw/develop/assets/logo.png) &nbsp;&nbsp; 1.0.0
==================================================================================================

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

Components
----------

Instead of providing a single method to solve a linear system, Ginkgo provides a selection of components that can be used to tailor the solver to your specific problem. It is also possible to use each component separately, as part of larger software. The provided components include matrix formats, solvers and preconditioners (commonly referred to as "_linear operators_" in Ginkgo), as well as executors, stopping criteria and loggers.

Matrix formats are used to represent the system matrix and the vectors of the system. The following are the supported matrix formats (__TODO:__ maybe add links, or better descriptions of the formats):

* `gko::matrix::Dense` - the row-major storage dense matrix format;
* `gko::matrix::Csr` - the Compressed Sparse Row (CSR) sparse matrix format;
* `gko::matrix::Coo` - the Coordinate (COO) sparse matrix format;
* `gko::matrix::Ell` - the ELLPACK (ELL_ sparse matrix format;
* `gko::matrix::Sellp` - the SELL-P sparse matrix format based on the sliced ELLPACK representation;
* `gko::matrix::Hybrid` - the hybrid matrix format that represents a matrix as a sum of an ELL and COO matrix.

All formats offer support for the `apply` operation that performs a (sparse) matrix-vector product between the matrix and one or multiple vectors. Conversion routines between the formats are also provided. `gko::matrix::Dense` offers an extended interface that includes simple vector operations such as addition, scaling, dot product and norm, which are applied on each column of the matrix separately.
The interface for all operations is designed to allow any type of matrix format as a parameter. However, version 1.0.0 of this library supports only instances of `gko::matrix::Dense` as vector arguments (the matrix arguments do not have any limitations).

Solvers are utilized to solve the system with a given system matrix and right hand side. Currently, you can choose from several high-performance Krylov methods implemented in Ginkgo:

* `gko::solver::Cg` - the Conjugate Gradient method (CG) suitable for symmetric positive definite problems;
* `gko::solver::Fcg` - the flexible variant of Conjugate Gradient (FCG) that supports non-constant preconditioners;
* `gko::solver::Cgs` - the Conjuage Gradient Squared method (CGS) for general problems;
* `gko::solver::Bicgstab` - the BiConjugate Gradient Stabilized method (BiCGSTAB) for general probles;
* `gko::solver::Gmres` - the restarted Generalized Minimal Residual method (GMRES) for general problems.

All solvers work with system matrices stored in any of the matrix formats described above, and any other general _linear operator_, such as combinations and compositions of other operators, or any matrix format you defined specifically for your application.

Preconditioners can be effective at improving the convergence rate of Krylov methods. All solvers listed above are implemented with preconditioning support. This version of Ginkgo has support for one preconditioner type, but stay tuned, as more preconditioners are coming in future releases:

* `gko::preconditioner::Jacobi` - a highly optimized version of the block-Jacobi preconditioner (block-diagonal scaling), optionally enhanced with adaptive precision storage scheme for additional performance gains.

You can use the block-Jacobi preconditioner with system matrices stored in any of the built-in matrix formats and any custom format that has a defined conversion into a CSR matrix.

As described in the "Designed for HPC" section, you have a choice between 3 different executors:

* `gko::CudaExecutor` - offers a highly optimized GPU implementation tailored for recent HPC systems;
* `gko::ReferenceExecutor` - single-threaded reference implementation for easy development and testing on systems without a GPU;
* `gko::OmpExecutor` - preliminary OpenMP-based implementation for CPUs.

With Ginkgo, you have fine control over the solver iteration process to ensure that you obtain your solution under the time and accuracy constraints of your application. Ginkgo supports the following stopping criteria out of the box:

* `gko::stop::Iteration` - the iteration process is stopped once the specified iteration count is reached;
* `gko::stop::ResidualNormReduction` - the iteration process is stopped once the initial residual norm is reduced by the specified factor;
* `gko::stop::Time` - the iteration process is stopped if the specified time limit is reached.

You can combine multiple criteria to achieve the desired result, and even add your own criteria to the mix.

Ginkgo also allows you to keep track of the events that happen while using the library, by providing hooks to those events via the `gko::log::Logger` abstraction. These hooks include everything from low-level events, such as memory allocations, deallocations, copies and kernel launches, up to high-level events, such as linear operator applications and completions of solver iterations. While the true power of logging is enabled by writing application-specific loggers, Ginkgo does provide several built-in solutions that can be useful for debugging and profiling:

* `gko::log::Convergence` - allows access to the final iteration count and residual of a Krylov solver;
* `gko::log::Stream` - prints events in human-readable format to the given output stream as they are emitted;
* `gko::log::Record` - saves all emitted events in a data structure for subsequent processing;
* `gko::log::Papi` - converts between Ginkgo's logging hooks and the standard PAPI Software Defined Events (SDE) interface (note that some details are lost, as PAPI can represent only a subset of data Ginkgo's logging can provide).

