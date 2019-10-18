![Ginkgo](https://raw.githubusercontent.com/ginkgo-project/ginkgo/develop/assets/logo.png)

About
-----

Welcome to the Ginkgo wiki pages! Ginkgo is a high-performance linear algebra library for manycore systems, with a focus on sparse solution of linear systems. It is implemented using modern C++ (you will need at least C++11 compliant compiler to build it), with GPU kernels implemented in CUDA. Currently, it runs on x86 and ARM CPUs, and Nvidia GPUs. It achieves high performance on GPUs, however the parallel CPU module is still not implemented (but coming soon!), so only a reference sequential implementation is available for CPUs. A multi-GPU module, as well as a hybrid CPU + multi-GPU module is planned in the future. Running across multiple nodes is not planned at this point.

Ginkgo is an academic project run by a group of researchers from several institutions: Karlsruhe Institute of Technology (Germany), Universitat Jaume I (Spain), and University of Tennessee (US). However, Ginkgo is not just about publishing academic papers. We care about building an open source library that will be used by the academic community to advance the understanding of sparse computations on modern hardware. We also care about building a high-quality library that can be used in comercial software. To facilitate both of these goals we try our best to design a modern and easy to use C++ interface, maintain extensive documentation for all parts of the library, cover the entire functionality with unit tests and provide a permissive 3-clause BSD licence.

In This Wiki
------------
If you are a new user and are facing some compilation issues, please see [FAQ](https://github.com/ginkgo-project/ginkgo/wiki/Frequently-asked-questions-(FAQ)). If you have a different problem/question, please feel free to create an issue with the tag _question_.

Here you will find a collection of resources related to the Ginkgo library. If you want to learn how to use Ginkgo, you can get started with [The Tutorial](./Tutorial:-Building-a-2D-Poisson-Solver) (__under construction__, for now, you can see the syllabus, and what is, or will be available in Ginkgo). If you want to contribute to Ginkgo, or are one of our developers, see the [Developers Homepage](./Developers-Homepage), and especially the [Contributing Guidelines](./Contributing-guidelines). If you think you found a bug in Ginkgo, please use the [Issue Tracker](/ginkgo-project/ginkgo/issues) to report it (please label the issue with _bug_). If you have feature requests or suggestions about the interface or the code, also use the [Issue Tracker](/ginkgo-project/ginkgo/issues), and use the _proposal_ label.

Contact Us
----------

If you have any questions, comments, or suggestions we would like to hear from you. Send us a message at one of the following e-mails:

Hartwig Anzt <hartwig.anzt@kit.edu>  
Terry Cojean <terry.cojean@kit.edu>  
Goran Flegar <flegar@uji.es>

