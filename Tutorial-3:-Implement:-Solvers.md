Previous: [Implement: Matrices](./Tutorial-2:-Implement:-Matrices); Next: [Optimize: Measuring Performance](./Tutorial-4:-Optimize:-Measuring-Performance)

Objectives
----------

This tutorial will explain how to solve a system of equations using Ginkgo. You will learn about `gko::solver::Cg` class and its accompanying factory `gko::solver::Cg::Factory`. You will also learn about stopping criteria (`gko::stop::Iterations` and `gko::stop::RelativeResidual`).

The CG solver
-------------

In case you are not familiar with the conjugate gradient method, you can read up on it [here](https://en.wikipedia.org/wiki/Conjugate_gradient_method).

In Ginkgo, to set up a cg solver, we first have to create a solver factory. That is done by 

```c++
    cg::build()
        .with_criteria(gko::stop::Iteration::build()
                           .with_max_iters(discretization_points)
                           .on(exec),
                       gko::stop::ResidualNormReduction<>::build()
                           .with_reduction_factor(1e-6)
                           .on(exec))
        .on(exec)
```

Here, we configure our solver to run on our `exec` executor and to have two stopping criteria.
The first one will simply stop after as many iterations as we have discretization points. 
The second one will stop if the residual norm decreases less than 1e-6 in some step.
We notice again that we need to pass the `exec` executor to all these settings: 
if we had not used
```c++
    const auto exec = gko::ReferenceExecutor::create();
```
to create `exec` as Reference executor on the CPU, but instead used
```c++
    const auto exec = gko::CudaExecutor::create(0, reference);
```
to create a Cuda executor
(as we will do in the [GPU Tutorial](./Tutorial-8:-Optimize:-Using-GPUs)), 
all these parameters and settings would be located on the GPU.

For iterative solvers that can accept a preconditioner, 
we can extend the solver factory with a precoditioner.
In the following code, we enhance the CG solver with a block-Jacobi preconditioner.
We note that we need to pass a copy of the system matrix to the block-Jacobi preconditioner
as the latter will in-place overwrite the matrix with its block inverse.
If we do not specify an upper blocksize limit (we use the limit `8` in the example below), the scalar Jacobi preconditioner is used.

```c++
    cg::build()
        .with_criteria(gko::stop::Iteration::build()
                           .with_max_iters(discretization_points)
                           .on(exec),
                       gko::stop::ResidualNormReduction<>::build()
                           .with_reduction_factor(1e-6)
                           .on(exec))
        .with_preconditioner(bj::build().with_max_block_size(8u).on(exec))
            .on(exec);
```
We will learn more sophisticated preconditioner usage in the [Preconditioner Tutorial](./Tutorial-7:-Optimize:-Using-a-Preconditioner).

Once the solver factory is generated, we can create an actual solver instance by binding it to a specfic target matrix:

```c++
    // Create solver
    auto solver = solver_gen->generate(A);
```
    

To finally solve our system we have to give the solver ownership of our right hand side and solution vector for the time it is working. This is done with `lend(vector)`. To apply the solver to these vectors, we just call `apply`:

```c++
    // Solve system
    solver->apply(lend(rhs), lend(u));
```

The solver will now write the calculated solution to our vector u.

Previous: [Implement: Matrices](./Tutorial-2:-Implement:-Matrices); Next: [Optimize: Measuring Performance](./Tutorial-4:-Optimize:-Measuring-Performance)
