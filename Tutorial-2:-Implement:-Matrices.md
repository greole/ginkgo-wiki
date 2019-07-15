Previous: [Getting Started](./Tutorial-1:-Getting-Started); Next: [Implement: Solvers](./Tutorial-3:-Implement:-Solvers)

Objectives
----------

In this tutorial you will learn the basics classes in Ginkgo. It will introduce you with the `gko::matrix::Dense` matrix class, and one of Ginkgo's core classes, the `gko::ReferenceExecutor`. We will cover how to create matrices, fill them with the coefficients and right hand side of the Poisson equation, create an initial guess vector, and compute the residual of the initial guess.

The Ginkgo Reference Executor
-----------------------------

In Ginkgo, there are three implementations of most functionalities: one reference implementation, one OpenMP implementation and one CUDA implementation.
The reference implementation is a stable, serial implementation against which the parallel and optimized implementations can be tested for correctness.
The OpenMP implentation uses OpenMP pragmas to parallelize the code on CPUs while the CUDA implementation provides a highly parallel implementation for NVIDIA-GPUs.

To specify which implementation should be used, Ginkgo uses so-called executors. In this step of the tutorial you will use only reference implementations, which are handled by the `gko::ReferenceExecutor`.
In order to create an executor, you only need to include the ginkgo header file:

```sh
#include <ginkgo/ginkgo.hpp>

int main(int argc, char *argv[]) 
{
    const auto reference = gko::ReferenceExecutor::create();
}
```

After creating an executor, you now can use it to set up your system matrix, right hand side and initial guess.

Creating a matrix in Ginkgo
---------------------------

In this step, you will create a dense matrix and fill it with the coefficients of the Poisson equation.
To create a matrix, you have to specify on which executor it should be created, which matrix format you want to use, which data type the entries should be and how large your matrix will be.
You will create a dense matrix with double entries and one row / column each for every discretization point of your Poisson problem. The following creates such a matrix on your reference executor:

```sh
#include <ginkgo/ginkgo.hpp>

int main(int argc, char *argv[]) 
{
    const unsigned int discretization_points = 100;

    using mtx = gko::matrix::Dense<double>;

    const auto reference = gko::ReferenceExecutor::create();

    auto matrix = mtx::create(reference, gko::dim<2>(discretization_points));
}
```

Filling the matrix with values
------------------------------


Previous: [Getting Started](./Tutorial-1:-Getting-Started); Next: [Implement: Solvers](./Tutorial-3:-Implement:-Solvers)
