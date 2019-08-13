Previous: [Implement: Matrices](./Tutorial-2:-Implement:-Matrices); Next: [Optimize: Measuring Performance](./Tutorial-4:-Optimize:-Measuring-Performance)

Objectives
----------

This tutorial will explain how to solve a system of equations using Ginkgo. You will learn about `gko::solver::Cg` class and its accompanying factory `gko::solver::Cg::Factory`. You will also learn about stopping criteria (`gko::stop::Iterations` and `gko::stop::RelativeResidual`).

The CG solver
-------------

In case you are not familiar with the conjugate gradient method, you can read up on it [here](https://en.wikipedia.org/wiki/Conjugate_gradient_method).

In Ginkgo, to set up a cg solver, we first have to create a solver factory. That is done by 

```c++
    auto solver_factory = cg::build()
                              .with_criteria(gko::stop::Iteration::build()
                                                 .with_max_iters(discretization_points)
                                                 .on(exec),
                                             gko::stop::ResidualNormReduction<>::build()
                                                 .with_reduction_factor(1e-6)
                                                 .on(exec))
                               .on(exec)
```

Here, we configure our solver to run on our `exec` executor and to have two stopping criteria.
The first stopping criteria asks the solver to stop after as many iterations as we have discretization points. 
The second criteria asks the solver to stop when the residual norm has reduced by a factor of 1e-6 from the initial residual. It should be noted that the solver stops when either one of these criteria is satisfied.

We notice again that we need to pass the `exec` executor to all these settings: 
if we had used
```c++
    const auto exec = gko::CudaExecutor::create(0, reference);
```
instead of
```c++
    const auto exec = gko::ReferenceExecutor::create();
```
to create a Cuda executor
(as we will do in the [GPU Tutorial](./Tutorial-8:-Optimize:-Using-GPUs)), 
all these parameters and settings (and hence the solver factory) would be located on the GPU.

For iterative solvers that can accept a preconditioner, 
we can extend the solver factory with a preconditioner.
In the following code, we enhance the CG solver with a block-Jacobi preconditioner.
If we do not specify an upper blocksize limit (we use the limit `8` in the example below), the scalar Jacobi preconditioner is used.

```c++
    using bj = gko::preconditioner::Jacobi<>;
    auto solver_factory = cg::build()
                             .with_criteria(gko::stop::Iteration::build()
                                                .with_max_iters(discretization_points)
                                                .on(exec),
                                            gko::stop::ResidualNormReduction<>::build()
                                                .with_reduction_factor(1e-6)
                                                .on(exec))
                             .with_preconditioner(bj::build().with_max_block_size(8u).on(exec))
                             .on(exec);
```
We will learn more about sophisticated preconditioner usage in the [Preconditioner Tutorial](./Tutorial-7:-Optimize:-Using-a-Preconditioner).

Once the solver factory is generated, we can create an actual solver instance by binding it to a specfic target matrix:

```c++
    // Create solver
    auto solver = solver_factory->generate(matrix);
```
    

To finally solve our system we have to give the solver ownership of our right hand side and solution vector for the time it is working. This is done with `gko::lend(vector)`. To apply the solver to these vectors, we just call `apply`:

```c++
    // Solve system
    solver->apply(gko::lend(rhs), gko::lend(u));
```

The solver will now write the calculated solution to our vector u.

Previous: [Implement: Matrices](./Tutorial-2:-Implement:-Matrices); Next: [Optimize: Measuring Performance](./Tutorial-4:-Optimize:-Measuring-Performance)
