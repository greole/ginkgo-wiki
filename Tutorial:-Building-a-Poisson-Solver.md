Welcome to Ginkgo tutorials! In this tutorial series you will build a simple console application which will be able to solve a discrete Poission equation on a unit square with Dirichlet boundary conditions. Your application will have support for several different right hand side and boundary functions, and will be able to run the solver either on the CPU or on the GPU. You will also be able to monitor the residual on screen as your solver progresses, and collect various metrics that you can later plot to analyse your results.

This series is divided into 4 sections:
1.  The [Getting Started](./Tutorial-1:-Getting-Started) section will guide you through Ginkgo's installation process.
2.  In the [Implement](./Tutorial-2:-Implement:-Matrices) section you will familiarize yourself with basic concepts in Ginkgo, such as matrices, solvers and linear operators, and learn enough of Ginkgo's features to build a first version of the Poisson solver.
3.  The [Optimize](./Tutorial-4:-Optimize:-Measuring-Performance) section will show you how to use Ginkgo's benchmarking and logging features to analize the performance of your solver, and show you how to improve the performance by using more suitable matrix formats and adding a preconditioner. You will also port your code to a GPU, to be able to use all the extra bandwidth it provides.
4.  The last [Customize](./Tutorial-9:-Customize:-Loggers) section will teach you about advanced concepts in Ginkgo and different ways in which Ginkgo can be extended for your applications specific need. In the first part of this section, you will improve your application with a custom stopping criterion and a custom logger, while the second part will show you how to squeeze the last bit of performance from your hardware by designing your own matrix format, solver and preconditioner which will benefit from the special structure of the Poisson matrix.

Without further ado, let's [get started!](./Tutorial-1:-Getting-Started)

Next: [Getting Started](./Tutorial-1:-Getting-Started)
