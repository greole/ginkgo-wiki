Previous: [Implement: Matrices](./Tutorial-2:-Implement:-Matrices); Next: [Optimize: Measuring Performance](./Tutorial-4:-Optimize:-Measuring-Performance)

Objectives
----------

This tutorial will explain how to solve a system of equations using Ginkgo. You will learn about `gko::solver::Cg` class and its accompanying factory `gko::solver::Cg::Factory`. You will also learn about stopping criteria (`gko::stop::Iterations` and `gko::stop::RelativeResidual`).

The CG solver
-------------

In case you are not familiar with the conjugate gradient method, you can read up on it [here](https://en.wikipedia.org/wiki/Conjugate_gradient_method).

In Ginkgo, to set up a cg solver, we first have to create a factory. That is done by 

```c++
cg::build()
		.with_criteria(gko::stop::Iteration::build()
						   .with_max_iters(max_iters)
						   .on(exec),
					   gko::stop::ResidualNormReduction<>::build()
						   .with_reduction_factor(1e-6)
						   .on(exec))
		.on(exec);

Previous: [Implement: Matrices](./Tutorial-2:-Implement:-Matrices); Next: [Optimize: Measuring Performance](./Tutorial-4:-Optimize:-Measuring-Performance)
