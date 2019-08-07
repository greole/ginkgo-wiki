Previous: [Implement: Solvers](./Tutorial-3:-Implement:-Solvers); Next: [Optimize: Monitoring Progress](./Tutorial-5:-Optimize:-Monitoring-Progress)

Objectives
----------

This tutorial will show you how to get basic performance measurements of your solver by using Ginkgo's benchmarking API. 

Measuring Performance
---------------------

As we have seen in the previous example, solving a system of equations consists of mainly two steps. The first one is the setup, where you create the `solver_factory` and `generate` the solver from the factory. The second step consists of the actual application of the solver to one or more right hand side vectors. The `apply` here is an operator application, which in the case of a solver is solving the system with the given right hand sides to get the solution.

Hence, in most cases, you will be generating the solver once, and using multiple "applies" to solve your right hand sides. Nevertheless, to be complete you should measure the performance of both the generate step and the apply step to understand your performance.

To measure the run-times of your solver you can use `std::chrono`. Consider the following snippet where we use the same CG solver as from the previous tutorial.

```c++
#include <ginkgo/ginkgo.hpp>

int main(){
.
.
.
    // Setup the solver factory with criteria and parameters
    auto solver_factory = cg::build()
                             .with_criteria(gko::stop::Iteration::build()
                                                .with_max_iters(discretization_points)
                                                .on(exec),
                                            gko::stop::ResidualNormReduction<>::build()
                                                 .with_reduction_factor(1e-6)
                                                 .on(exec))
                             .with_preconditioner(bj::build().with_max_block_size(8u).on(exec))
                             .on(exec);
    
    
    // Synchronize before timing.
    exec->synchronize();
    auto g_tic = std::chrono::steady_clock::now();
    // Create solver factory
    auto solver = solver_factory->generate(A);
    // Synchronize before timing.
    exec->synchronize();
    auto g_tac = std::chrono::steady_clock::now();
    auto generate_time =
         std::chrono::duration_cast<std::chrono::nanoseconds>(g_tac -
                                                              g_tic);
     
    exec->synchronize();
    auto a_tic = std::chrono::steady_clock::now();
    // Solve system
    solver->apply(gko::lend(b), gko::lend(x));
    exec->synchronize();
    auto a_tac = std::chrono::steady_clock::now();
    auto apply_time =
         std::chrono::duration_cast<std::chrono::nanoseconds>(a_tac -
                                                              a_tic);
    
}
```
You can see mainly three blocks. The first blocks sets up the solver with the criteria and parameters with the preconditioner if required, the second block measures the performance of the generation of the solver itself from the factory. This generate step involves the initialization of the variables and also involves the generation of the preconditioner if needed from the system matrix, A. The final block measures the performance of the apply step of the solver, which as discussed in the previous Tutorial actually solves the system.

Some notes
----------

You can see that we always call `synchronize()` before calling the timer. This is to make sure that we are accurate that no other stray tasks pollute our timings and in general it is a good idea to do this especially in the case of the CUDA executor to get accurate timings. 



Previous: [Implement: Solvers](./Tutorial-3:-Implement:-Solvers); Next: [Optimize: Monitoring Progress](./Tutorial-5:-Optimize:-Monitoring-Progress)