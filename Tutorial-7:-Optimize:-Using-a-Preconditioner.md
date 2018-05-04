Objectives
----------

After changing the matrix format and significantly increasing the performance of each iteration of your solver, in this tutorial we will try to decrease the number of iterations our solver needs to converge by using a preconditioner. You will learn about `gko::preconditioner::BlockJacobi`, `gko::pattern::BlockIdentification` and `gko::pattern::SupervariableAgglomeration`, as well as their respective factories.