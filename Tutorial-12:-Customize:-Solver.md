Objectives
----------

In this tutorial you will create your own specialized version of Cg that exploits the special matrix structure of the Poisson matrix to get even better performance. You will learn how Ginkgo uses the abstract `gko::LinOp` interface to be able to support a wide variate of combinations of solvers, preconditioners and matrix formats.