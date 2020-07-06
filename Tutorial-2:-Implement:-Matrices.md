Previous: [Getting Started](./Tutorial-1:-Getting-Started); Next: [Implement: Solvers](./Tutorial-3:-Implement:-Solvers)

Objectives
----------

In this tutorial you will learn about the basic classes of Ginkgo. We will introduce Ginkgo's executor concept and the `gko::matrix::Dense` matrix class. 
Precisely, we will learn how to create matrices and fill them with the coefficients. 
Similarly, we will learn about how to create a right-hand-side for the Poisson equation, create an initial guess vector, and compute the residual of the initial guess.

The Ginkgo Reference Executor
-----------------------------

Ginkgo radically separates the algorithms from the hardware-specific kernel realizations.

Currently, there are three kernel realizations available in Ginkgo: 
The `reference` kernels are designed as bullet-proof sequential kernels that are 
guaranteed to compute the correct solution.
Their primary purpose is to ensure the correctness of complex algorithms, 
and to compute the (exec) solutions in the unit tests checking the correctness of
the performance-optimized kernels.
The `omp` kernels employ OpenMP pragmas to leverage the compute power of 
multiprocessors such as Multicore CPUs.
The `cuda` kernels are implemented in the NVIDIA-specific CUDA programming language 
and heavily optimized for efficient usage of NVIDIA GPUs.

To specify which kernel implementation should be used, 
Ginkgo uses so-called `executors` that are passed to every function and routine call. 
In this step of the tutorial we will use only the reference implementations, 
which are handled by the `gko::ReferenceExecutor`.
In order to create an executor in general and the reference executor in particular, 
we need to include the ginkgo header file and create the executor:

```c++
#include <ginkgo/ginkgo.hpp>

int main(int argc, char *argv[]) 
{
    const auto exec = gko::ReferenceExecutor::create();
}
```
After creating an executor, we can use it for the remainder of the program, to invoke the executor-specific kernels.


In the following, we use the created Reference executor to set up our system matrix, right hand side and initial guess.

Creating a matrix in Ginkgo
---------------------------

In this step, we will create a dense matrix and fill it with the coefficients of the Poisson equation.
To create a matrix, we have to specify on which executor it should be created, which matrix format we want to use, which data type the entries should be and how large our matrix will be.
We will create a dense matrix with double entries and one row / column each for every discretization point of our Poisson problem. The following creates such a matrix on our (exec) executor:

```c++
#include <ginkgo/ginkgo.hpp>

int main(int argc, char *argv[]) 
{
    const unsigned int discretization_points = 100;

    using mtx = gko::matrix::Dense<double>;

    const auto exec = gko::ReferenceExecutor::create();

    auto matrix = mtx::create(exec, gko::dim<2>(discretization_points));
}
```
It is important to pass the executor to the matrix creation function. This becomes obvious when considering that passing a Cuda executor results in the matrix being created on the CUDA-capable GPU.

While the code above creates a dense matrix, replacing
```c++
    using mtx = gko::matrix::Dense<double>;
```
with
```c++
    using mtx = gko::matrix::Csr<double>;
```
or
```c++
    using mtx = gko::matrix::Coo<double>;
```
would create a sparse matrix of [CSR](./Matrix-Formats-in-Ginkgo.md#gkomatrixcsr) and [COO](./Matrix-Formats-in-Ginkgo.md#gkomatrixcoo) type, respectively.

We also note that `gko::dim<2>(discretization_points)` creates a square matrix of size `discretization_points x distretization_points`. 
Larger dimension parameters (used in the angle brackets) will result in
the creation of tensors.
We will later see how to create non-square matrices.

Filling the matrix with values
------------------------------

Now, let us fill the matrix with the three point stencil for the Poisson equation. We will do this using the following function:

```c++
void generate_stencil_matrix(gko::matrix::Dense<> *matrix)
{
    const auto discretization_points = matrix->get_size()[0];
    const double coefs[] = {-1, 2, -1};
    for (int i = 0; i < discretization_points; ++i) {
        for (auto dofs : {-1, 0, 1}) {
            if (0 <= i + dofs && i + dofs < discretization_points) {
            	matrix->at(i, i + dofs) = coefs[dofs + 1];
            }
        }
    }
}
```

To get `generate_stencil_matrix` to actually fill the matrix, we have to temporarily hand over ownership (smart pointers) of the matrix and get it back once the matrix is all set up. This is done by [`gko::lend(matrix)`](https://ginkgo-project.github.io/ginkgo/doc/develop/namespacegko.html#aa8cb4876b72e5e1036ea9575443c439b):

```c++
#include <ginkgo/ginkgo.hpp>

int main(int argc, char *argv[]) 
{
.
.
.
    auto matrix = mtx::create((exec), gko::dim<2>(discretization_points));
    
    generate_stencil_matrix(gko::lend(matrix));
}
```

Creating the right hand side and an initial guess vector
--------------------------------------------------------

We will use u(x) = x^3 as a known solution, so for the right hand side we get f(x) = 6x. Now, we have to create vectors for the solution and the right hand side and fill them with values. 
We can handle vectors as dense matrices, i.e. create the vectors as an instance of the dense matrix class (with the `using vec = gko::matrix::Dense<double>`). For example, a single vector is stored as a dense matrix of dimension `n x 1` (n rows and 1 column). This allows us to handle systems with multiple right hand sides as well.

As an initial guess, we will just use a vector filled with zeros. For the right hand side, we evaluate f at our discretization points and get the desired values.

```c++
#include <ginkgo/ginkgo.hpp>

int main(int argc, char *argv[]) 
{
.
.
.
    using vec = gko::matrix::Dense<double>;
    auto correct_u = [](double x) { return x * x * x; };
    auto f = [](double x) { return 6 * x; };
    auto u0 = correct_u(0);
    auto u1 = correct_u(1);

    auto rhs = vec::create(exec, gko::dim<2>(discretization_points, 1));
    generate_rhs(f, u0, u1, lend(rhs));
    auto u = vec::create(exec, gko::dim<2>(discretization_points, 1));
    for (int i = 0; i < u->get_size()[0]; ++i) {
        u->get_values()[i] = 0.0;
    }
}
```
We use lambda functions to set up our required mathematical functions and you see that the `generate_rhs` takes in these lambdas and computes the values of the vectors at each of the discretization points as required.

The function `generate_rhs` takes the lambda function and computes the `rhs` vector. 

```c++
template <typename Closure>
void generate_rhs(Closure f, double u0, double u1, gko::matrix::Dense<> *rhs)
{
    const auto discretization_points = rhs->get_size()[0];
    auto values = rhs->get_values();
    const auto h = 1.0 / (discretization_points + 1);
    for (int i = 0; i < discretization_points; ++i) {
        const auto xi = (i + 1) * h;
        values[i] = -f(xi) * h * h;
    }
    values[0] += u0;
    values[discretization_points - 1] += u1;
}
```

In the next step, we will set up a solver and solve the system we just set up.

Previous: [Getting Started](./Tutorial-1:-Getting-Started); Next: [Implement: Solvers](./Tutorial-3:-Implement:-Solvers)
