![Ginkgo](https://github.com/ginkgo-project/ginkgo/raw/develop/assets/logo.png)

The Ginkgo team is proud to announce the first release of Ginkgo, the next-generation high-performance sparse linear algebra library. Ginkgo leverages the features of modern C++ to give you a tool for the iterative solution of linear systems that is:

* __Easy to use.__ Interfaces with cryptic naming schemes and dozens of parameters are a thing of the past. Ginkgo was build with good software design in mind, making simple things simple to express.
* __High performance.__ Our optimized CUDA kernels ensure you are reaching the potential of today's GPU-accelerated high-end systems.
* __Controllable.__  While Ginkgo can automatically moves your data when needed, you remain in control by optionally specifying when the data is moved and what is its ownership scheme.
* __Composable.__ Iterative solution of linear systems is an extremely versatile field, where effective methods are built by mixing and match various components. Need a GMRES solver preconditioned with a block-Jacobi enhanced BiCGSTAB? Thanks to its novel linear operator abstraction, Ginkgo can do it!
* __Extensible.__ Did not find a component you were looking for? Ginkgo is designed to be easily extended in various ways. You can provide your own loggers, stopping criteria, matrix formats, preconditioners and solvers to Ginkgo and have them integrate as well as the natively supported ones, without the need to modify or recompile the library.

