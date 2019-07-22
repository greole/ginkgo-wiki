Previous: [Getting Started](./Tutorial-1:-Getting-Started); Next: [Implement: Solvers](./Tutorial-3:-Implement:-Solvers)

Objectives
----------

In this tutorial you will learn the basics classes in Ginkgo. It will introduce you with the `gko::matrix::Dense` matrix class, and one of Ginkgo's core classes, the `gko::ReferenceExecutor`. We will cover how to create matrices, fill them with the coefficients and right hand side of the Poisson equation, create an initial guess vector, and compute the residual of the initial guess.

The Ginkgo Reference Executor
-----------------------------

In Ginkgo, there are three implementations of most functionalities: one reference implementation, one OpenMP implementation and one CUDA implementation.
The reference implementation is a stable, serial implementation against which the parallel and optimized implementations can be tested for correctness.
The OpenMP implentation uses OpenMP pragmas to parallelize the code on CPUs while the CUDA implementation provides a highly parallel implementation for NVIDIA-GPUs.

To specify which implementation should be used, Ginkgo uses so-called executors. In this step of the tutorial we will use only reference implementations, which are handled by the `gko::ReferenceExecutor`.
In order to create an executor, we only need to include the ginkgo header file:

```sh
#include <ginkgo/ginkgo.hpp>

int main(int argc, char *argv[]) 
{
    const auto reference = gko::ReferenceExecutor::create();
}
```

After creating an executor, we now can use it to set up our system matrix, right hand side and initial guess.

Creating a matrix in Ginkgo
---------------------------

In this step, we will create a dense matrix and fill it with the coefficients of the Poisson equation.
To create a matrix, we have to specify on which executor it should be created, which matrix format we want to use, which data type the entries should be and how large our matrix will be.
We will create a dense matrix with double entries and one row / column each for every discretization point of our Poisson problem. The following creates such a matrix on our reference executor:

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

Now, let us fill the matrix with the three point stencil for the Poisson equation. We will do this using the following function:

```sh
void generate_stencil_matrix(gko::matrix::Dense<> *matrix)
{
    const auto discretization_points = matrix->get_size()[0];
    const double coefs[] = {-1, 2, -1};
    for (int i = 0; i < discretization_points; ++i) {
        for (auto ofs : {-1, 0, 1}) {
            if (0 <= i + ofs && i + ofs < discretization_points) {
            	matrix->at(i, i + ofs) = coefs[ofs + 1];
            }
        }
    }
}
```

To get `generate_stencil_matrix` to actually fill the matrix, we have to temporarily hand over ownership of the matrix and get it back once the matrix is all set up. This is done by `lend(matrix)`:

```sh
#include <ginkgo/ginkgo.hpp>

int main(int argc, char *argv[]) 
{
.
.
.
    auto matrix = mtx::create(reference, gko::dim<2>(discretization_points));
    
    generate_stencil_matrix(lend(matrix));
}
```

Creating the right hand side and an initial guess vector
--------------------------------------------------------

We will use 

Previous: [Getting Started](./Tutorial-1:-Getting-Started); Next: [Implement: Solvers](./Tutorial-3:-Implement:-Solvers)
