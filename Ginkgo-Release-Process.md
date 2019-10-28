The purpose of this page is to elaborate a safe procedure for releasing Ginkgo code. The aim is to cover important topics for releasing the code and ensuring there is less mistakes when going through this process by following a sort of checklist we could come up with.

This page contains the following steps:
1. [Preliminary procedures](#preliminary-procedures)
2. [Building binaries using CPack](#building-binaries-using-cpack)
3. [Publishing the Release](#publishing-the-release)
4. [Post-releasing process](#post-releasing-process)

Preliminary procedures
----------------------
This is highly recommended to ensure a good quality of the Ginkgo releases.
In general, these steps relate to making sure the release process goes as smoothly as possible and that we reduce the amount of problems for our users and contributors as much as possible.

### Creating a release Milestone and Issue
In order to assist in the release process, and center the team's focus towards the release, it is important to create a new milestone for the release and an issue to manage the release process. At some point in time it should be possible to add an issue template for a release-type of issue, but we do not have one yet.

To create a milestone, click on the milestone link from the issues's page, or [go here](https://github.com/ginkgo-project/ginkgo/milestones). The process is fairly simple and needs to further comments. Once this is done, add the newly created milestone to every issue and pull request which should be part of the release.

As issues are closed and pull requests are merged, the release status will gradually approach 100%. Whenever all issues are completed (and the milestone status reaches 100%), the remaining steps of the release process can be completed. Some of the following steps can still be started in parallel, but make sure to retest once all merge requests are merged!

### Manage open bugs
We currently have the tag "bug" in order to track detected bugs in Ginkgo. The following links allow you to see the list of bugs:
+ [Ginkgo open Gitlab bugs](https://gitlab.com/ginkgo-project/ginkgo/issues?scope=all&utf8=%E2%9C%93&state=opened&label_name[]=Bug)
+ [Ginkgo open Github bugs](https://github.com/ginkgo-project/ginkgo/issues?q=is%3Aissue+is%3Aopen+label%3ABug)

Before releasing, it is important to deal with bugs is some way. The best way to deal with bugs is to fix them before releasing -- if that is possible. This depends on the time constraints and the complexity of the bug. If the bug cannot be fixed in time for the release, then add to the [Known Issues page](https://github.com/ginkgo-project/ginkgo/wiki/Known-Issues) the existing bugs that we are aware of. It can be as simple as a link to the related issue but could also take a sentence for context and severity.

### Consider creating a release branch
When a lot of activity is happening in the develop branch, it is possible to use [the Gitflow Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) to prepare the release. The document itself describes the precise process, here is a summary of why it is useful and what it does.

The Gitflow Workflow defines a strict branching model designed around the project release. What it does is that it adds specific roles to different branches and defines how and when they should interact.

The base concept is to create a specific release branch, such as `release/1.0.0` where the release can be stabilized, all important pull requests and fixes can be integrated, and the release-specific tests can be executed. Once the process is completed, the release branch can be integrated into the master branch, and the release process can be started from the master branch.

### Update the external (third party) libraries
Currently, Ginkgo relies on set versions of third_party tools such as GTest, gflags, RapidJSON and more. The reason for doing this instead of relying on the last commits is that some important changes may happen in the third party package which may suddenly break Ginkgo's usage.

Before proceeding to a new release, it is important to update the external modules to a more recent version. For this process, first check if any new releases are available, then check for any important bug fixes or API changes notified in the software's release notes. Finally, if everything looks correct, try to update the external module. All tests and all code which use the third party tools should be ran on a variety of architectures (if possible, all supported architectures) to ensure that there is no unexpected problems. When no unexpected problems were found the third party packages update can be confirmed. 

### Check the quality of the release's tests and memory access
Before releasing, all tests should be run on as many platforms as possible using CI, but if possible it should also be ran on different architectures which we should support such as MacOS and Windows. Once normal tests all pass, it is required to check the memory accesses of the release and solve any important or easy to fix small bug, in particular identified by clang-tidy, iwyu, sonarqube, etc.

Thanks to CMake, we have support for inspecting our test suite with the `valgrind` tool. It is important to check that there are no major memory access issues before releasing. Failing to do so may cause severe problems for our users.

To run these tests, use the following procedure. First build Ginkgo as usual, ensure all flags being activated.Then launch the tests and finally the memchecks:
```bash 
# In the ginkgo folder
cmake  -G "Unix Makefiles" -H. -BDebug -DCMAKE_BUILD_TYPE=Debug -DGINKGO_DEVEL_TOOLS=ON \
      -DGINKGO_BUILD_TESTS=ON -DGINKGO_BUILD_REFERENCE=ON -DGINKGO_BUILD_OMP=ON \
      -DGINKGO_BUILD_CUDA=ON
cd Debug
ctest -T build -T test -T memcheck -T submit
```

This will output a summary of the test results in the command line: memcheck passing or not, and the amount of defects. The details are available in a directory such as `Testing/<timestamp>` and `Testing/Temporary`. Among other files, there should be number of files in the form of `MemoryChecker.xx.log`. Go through all of them and check which tests returned errors of which kind (error code).

The last target, `-T submit` submits the test to the [CDash dashboard](https://my.cdash.org/index.php?project=Ginkgo+Project). This should allow a nice visualization of the results.

Another alternative is to use the `cmake/CTestScript.cmake` which has its own documentation and allows by passing the correct parameters, to check coverage, run valgrind tests, run sonarqube tests, and more.

### Write release notes
It is important to write proper release notes. This will tell the users what they can expect from using this release. There are multiple aspects release notes should include:
+ A summary of the release, in a couple of sentence, what is in the release.
+ What are the system requirements, which systems are known to not be supported, what is unsupported.
+ Provide a link to the known issues.
+ List all changes, eventually sorted in different categories:
   + additions,
   + removals,
   + changes (such as interface, if any),
   + fixes.


In general, when writing release notes, try to stay concise and factual. Do not add jokes or unnecessary comments. A bullet-proof strategy to write the release changes is to go through every single pull request and summarize it in one line. If multiple pull requests touch to a broader set of changes, then it is fine to summarize them as a single bullet-point. Finally, it may be interesting for the users here to tag all pull requests linked to each particular point (simply by adding the `#number` next to it).

Building binaries using CPack
-----------------------------
This step is optional since our primary release mode is source mode. Nonetheless, it may be useful to release binaries for some mainstream architectures for convenience (Linux x86_64 and MacOS).

Thankfully, the CPack tool from CMake and Ginkgo's CMake setup greatly simplifies the creation of binary packages. CPack supports multiple `generators` for generating packages for different formats. An extensive documentation of the available package generators can be found [here](https://gitlab.kitware.com/cmake/community/wikis/doc/cpack/PackageGenerators). The CPack module documentation can be found [here](https://cmake.org/cmake/help/v3.9/module/CPack.html). Some interesting packages generators are:
+ TGZ, ZIP
+ DEB for Debian like distributions
+ RPM for Red Hat based distributions
+ For MacOs, there are multiple tools supported such as: DragNDrop, PackageManager, OSXX11 and Bundle.

The general process for creating a package which contains both, a release and a debug version of Ginkgo, is:
1. Prepare a release configuration of Ginkgo in one folder (call it `Release`).
2. In another folder, prepare a debug configuration of Ginkgo (call it `Debug`).
3. Compile both branches, launch tests and ensure everything passes.
4. Use the CPack command to generate packages in various formats.

All in all, the full process may look like this:
```bash
cmake  -G "Unix Makefiles" -H. -BRelease -DCMAKE_BUILD_TYPE=Release -DGINKGO_DEVEL_TOOLS=ON \
      -DGINKGO_BUILD_TESTS=ON -DGINKGO_BUILD_REFERENCE=ON -DGINKGO_BUILD_OMP=ON \
      -DGINKGO_BUILD_CUDA=ON
cmake  -G "Unix Makefiles" -H. -BDebug -DCMAKE_BUILD_TYPE=Debug -DGINKGO_DEVEL_TOOLS=ON \
      -DGINKGO_BUILD_TESTS=ON -DGINKGO_BUILD_REFERENCE=ON -DGINKGO_BUILD_OMP=ON \
      -DGINKGO_BUILD_CUDA=ON
pushd Release
ctest -T build -T test -T submit
popd
pushd Debug
ctest -T build -T test -T submit
popd
cpack -G TGZ -G ZIP -G<other formats>
```

Publishing the Release
----------------------
Once the release milestone is complete, all tests pass properly, important bugs are fixed or listed as known issues, the memory checks are correct, and the release notes are ready, the actual release can finally be published. The first step before starting the process is to decide on the new version's number by using the [semantic versioning guidelines](https://semver.org/), which is of the form `MAJOR.MINOR.PATCH`. When the number is decided, it is necessary to update our main `.gitlab-ci.yml` file to add to the gh-pages entry the tag version under `only: tags`, for more information see [this section](#documentation-and-website-update).

This process is simplifiied thanks to the respective [Github](https://help.github.com/articles/creating-releases/) and [Gitlab](https://docs.gitlab.com/ee/workflow/releases.html) release processes. Both processes work similarly. In general, we should be using only the Github one because our Gitlab page is private, except for the mirrored ``develop'' branch.

The general process is only to create a [git tag](https://git-scm.com/book/en/v2/Git-Basics-Tagging) with some information attached to it. This information should at least include the compiled release notes, and, if available, links to the compiled binaries (both Github and Gitlab allow you to update compiled binaries). In Github, the page is accessible [from the repository's main page's release tab](https://github.com/ginkgo-project/ginkgo/releases). It is required to add the tag as a "protected tag" in the repository as soon as it is published for proper CI system execution.

__TODO__ we should probably ensure to add a signing key for the release.

Post-releasing process
----------------------
After publishing the release, the full process in not yet completed. It is important to ensure the master branch is correctly updated to the new release, and that the documentation is properly generated and available.

### Updating the master branch
The master branch needs to be updated to the newly created tagged version. The process here uses standard [git branching and merging commands](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging), but it is important to get it right. It is important to:
1. Ensure a merge commit **is created**,
2. Edit the commit message to include the release notes and any other useful information.

```bash
git checkout master
git merge --no-ff --edit <tagged version>
# proceed to integrating the release notes
```
Another option worth considering is `--signoff`.

In fact, if the tag was properly added, no options are needed, but are useful to make sure that the release note is integrated and a merge commit was generated.

Once this step is completed, it is required necessary to move all of this to a new branch and create a new pull request as the master branch cannot be updated directly.

### Documentation and website update
For the documentation to be generated for the tagged version, the `gh-pages` test should be updated to contain the newly created tag, for example:
```yaml
gh-pages:
  stage: deploy
  [...] # various commands are skipped
  dependencies: []
  only:
    refs:
      - develop
      - master
    tags:
      - 1.0.0
      - <new version>
    variables:
      - $PUBLIC_CI_TAG
  except:
      - schedules
```

Once the tag is present, a [new pipeline](https://gitlab.com/ginkgo-project/ginkgo-public-ci/pipelines) can be invoked for the tagged version. This should generate the documentation for the release in the form: `https://ginkgo-project.github.io/ginkgo/doc/<version>`.