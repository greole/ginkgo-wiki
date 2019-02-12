The purpose of this page is to elaborate a safe procedure for releasing Ginkgo code. The aim is to cover important topics for releasing the code and ensuring there is less mistakes when going through this process by following a sort of checklist we could come up with.

Preliminary procedures
----------------------
This is highly recommended to ensure a good quality of the Ginkgo releases.
These steps in general relate to making sure the release process goes as smoothly as possible and that we reduce the amount of problems for our users and contributors as much as possible.

### Creating a release Milestone and Issue
In order to assist in the release process and center the team's focus towards the release, it is important to create a new milestone for the release and an issue to manage the release process. At some point in time it should be possible to add an issue template for a release-type of issue, but we do not have one yet.

To create a milestone, click on the milestone link from the issues's page, or [go here](https://github.com/ginkgo-project/ginkgo/milestones). The process is fairly simple and needs to further comments. Once this is done, add the newly created milestone to every issue and pull requests which should be part of the release.

The way milestone work, as issues are closed and pull requests merged it will gradually get closer to 100%. Whenever all other issues are finished and the milestone reaches 100%, it is time to proceed with the rest of the release process. Some of the following steps can still be started in parallel, but be sure to retest them when all the merge requests are merged!

### Manage open bugs
We currently have the tag "bug" in order to track bugs present in Ginkgo. The following links allow you to see the list of bugs:
+ [Ginkgo open Gitlab bugs](https://gitlab.com/ginkgo-project/ginkgo/issues?scope=all&utf8=%E2%9C%93&state=opened&label_name[]=Bug)
+ [Ginkgo open Github bugs](https://github.com/ginkgo-project/ginkgo/issues?q=is%3Aissue+is%3Aopen+label%3ABug)

Before releasing, it is important to deal with bugs is some way. The first obvious way is to fix them before releasing if that is possible. This depends on the time constraints and the complexity of the bug. If the bug cannot be fixed in time for the release, then add to the [Known Issues page](https://github.com/ginkgo-project/ginkgo/wiki/Known-Issues) the existing bugs that we know of. It can be as simple as a link to the relating issue but could also take a sentence for context and severity.

### Check the quality of the release's memory access
We have support for calling our test suite with the `valgrind` tool thanks to CMake. It is important to check that there is no major memory access issues before releasing. Failing to do so may cause severe problems for our users.

To run these tests, use the following procedure. First build Ginkgo as usual, ensure everything is on. Then launch the tests and finally the memchecks one :
```bash 
# In the ginkgo folder
cmake  -G "Unix Makefiles" -H. -BDebug -DCMAKE_BUILD_TYPE=Debug -DGINKGO_DEVEL_TOOLS=ON \
      -DGINKGO_BUILD_TESTS=ON -DGINKGO_BUILD_REFERENCE=ON -DGINKGO_BUILD_OMP=ON \
      -DGINKGO_BUILD_CUDA=ON
cd Debug
ctest -T build -T test -T memcheck -T submit
```

This will output a summary of the test results (memcheck passing or not and the amount of defects) in the command line. The details are available in a directory such as `Testing/<timestamp>` and `Testing/Temporary`. There should be among other files a lot of files in the form of `MemoryChecker.xx.log`. Go through all of them and check which sort of errors were returned and for which tests.

The last target, `-T submit` submits the test to the [CDash dashboard](https://my.cdash.org/index.php?project=Ginkgo+Project). This should allow a nice visualization of the results.

Building binaries using CPack
-----------------------------
This step is optional since our primary release mode is source mode. Nonetheless, it may be useful to release binaries for some of our important systems (Linux x86_64 and MacOS) as convenience.

Thankfully, the CPack tool from CMake and Ginkgo's CMake setup greatly simplifies the creation of binary packages. CPack supports multiple `generators` for generating packages for different formats. An extensive documentation of the available package generators can be found [here](https://gitlab.kitware.com/cmake/community/wikis/doc/cpack/PackageGenerators). The CPack module documentation can be found [here](https://cmake.org/cmake/help/v3.9/module/CPack.html). Some interesting packages generators are:
+ TGZ, ZIP
+ DEB for Debian like distributions
+ RPM for Red Hat based distributions
+ For MacOs, there are multiple tools supported such as: DragNDrop, PackageManager, OSXX11 and Bundle.

The general process is the following, but can be adapted:
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
cmake --build Release -j <nprocs>
cmake --build Debug -j <nprocs>
pushd Release
ctest -T test -T submit
popd
pushd Debug
ctest -T test -T submit
popd
cpack -G TGZ,ZIP,<other formats>
```