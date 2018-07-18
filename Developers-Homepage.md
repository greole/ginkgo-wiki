__Under construction__

Resources
---------

* A mandatory read for all new developers / contributors: [Contributing guidelines](Contributing-guidelines), still under construction (but you should read it anyway - and help completing it).
* A document describing the general design of Ginkgo: [Library design](Library-design). If you're not sure how something in Ginkgo works, or why something was done in a certain way, start here. If there's no answer in the document, feel free to ask. Once you've found your answer (either by asking, or by looking at the code in more detail), we encourage you to update the document with what you've found.
* A guide for reviewing pull requests can be found [here](Instructions-for-pull-request-(PR)-reviewers).
* If you're interested in how RTTI works, there's an informal description, with links to formal documents [here](Information-about-RTTI-(RunTime-Type-Information)).
* A funny slideshow on how to [write good commit messages](https://www.slideshare.net/TarinGamberini/commit-messages-goodpractices). Most of the tools are designed around these conventions, so it will make all our lives a bit easier if everyone follows that. If you don't like cats, you can read [this](https://gist.github.com/robertpainsi/b632364184e70900af4ab688decf6f53) instead (but feel free to read both).
* If by any chance you're still not using `BasicLinOp`, [here](Updating-linear-operators-to-use-BasicLinOp---instead-of-LinOp) is how you can convert your `LinOp`-based matrix/solver/preconditioner/whatever and simplify your life.
* A tutorial about template metaprogramming can be found [here](./Introduction-to-Template-Metaprogramming).

### Lvalue vs Rvalue References, Move Semantics and Perfect Forwarding
If you are interested in these topics, you should start by reading [Thomas Becker's article on the subject](http://thbecker.net/articles/rvalue_references/section_01.html) in 11 sections which introduce you to all of these subjects. Sections 4, 5 and 9 are important for efficient use of these concepts (you could make your code very inefficient otherwise).

In [Scott Meyers' article](https://accu.org/var/uploads/journals/Overload111.pdf) you will learn that `&&` does not always mean rvalue-references, notably due to template reference collapsing rules (seen in section 8 of Thomas Becker's article). The same topic is covered in a [video](https://channel9.msdn.com/Shows/Going+Deep/Cpp-and-Beyond-2012-Scott-Meyers-Universal-References-in-Cpp11) if you prefer that format.

A very simple and schematic way to view rvalues is an additional tool to standard references allowing pointer exchange through the reference system. `std::move(x)` converts its argument to an rvalue, and perfect forwarding (i.e. `std::forward<X>(a)` is really about returning the proper reference to what `a` actually is (rvalue or lvalue).