What to check
-------------

1. Verify that the PR does something meaningful, and that it actually does what the description says it does.
2. Make sure the code style conforms to [Contributing Guidelines](Contributing-guidelines).
3. Make sure that the public interface is documented with Doxygen comments.
4. Check that none of the configuration scripts have been modified (`.gitlab-ci.yml`, `.clang-format`, `.gitignore`), unless there was a good reason for it (that reason should be a part of the PR description).
5. Go over the code and point out any design issues, hard-to-understand methods, etc.
6. Make sure that all new features are adequately covered by unit tests.

How to run the CI
-----------------

For internal PRs coming from the repository itself the CI builds will be run automatically. Look for the _Checks_ checkmark above the merge button and make sure it is green.

For external PRs coming from other forks, you'll have to trigger the pipelines manually - __make sure the PR is not trying to do anything malicious before doing that!__
Go to the [Run pipeline dialog](https://gitlab.com/ginkgo-project/ginkgo-public-ci/pipelines/new) in the public gitlab CI. Set the __Create for__ field to `check_pr`, and under __Variables__ add a variable with key `PR_ID` and set the value to the ID of the PR you want to check. (e.g. for PR #42 you should set the value to `42`).
The special job from the `check_pr` branch will create a new branch from the PR on Gitlab CI, which will cause the pipelines to be run for it. When you return to the PR on Github, you should see the __Checks__ checkmark in the PR.

How to merge
------------

Ginkgo has two merge methods enabled: "_merge_" and "_squash and merge_". You should use the appropriate method depending on how the history of the pull request looks like:

*   If the history is nice, clean and linear, use _merge_. This will preserve the entire history for future reference.
*   If there's something crazy going on (50 commits for 3 lines of changes, merge commits within the feature branch, commits that remove changes introduced by other commits, etc.), just use "_squash and merge_". This will destroy the history of the pull request, but will keep the history of Ginkgo clean.

Make sure you __edit the commit message__ of the merge request. The default Github's message "Merge pull request #2982 from project-name/branch-name" is not very useful. There's no other way to get a commit in `develop`/`master` other than by merging a pull request and the PR number doesn't tell you much if you don't have access to github later. In addition, both Github and Gitlab (where the private fork of Ginkgo is) use hashes to denote issues, so using them in commit messages adds confusing back-references to issue trackers. Instead, you should describe in short what you are merging (e.g. "Merge CG reference kernels"), and add a __full link__ to the related pull request in the description section of the commit message. Make sure that the first line (i.e. the title) of the commit message is meaningful on its own and does not exceed 50 characters. Consider copying and reformatting the pull request description and including it as part of the commit message. Make sure the commit message does not exceed 72 characters per line (the only exception is a link longer than 72 characters), and has no Github-specific links (they should either be removed or replaced with full links - e.g. issue `#1` should be replaced with `https://github.com/ginkgo-project/ginkgo/issues/1`).

Finally, __do not merge PRs that are assigned to other people__, unless you have their explicit consent. Even if the PR is green (has been approved, passed the CI tests), it does not mean that the assignee is not doing some final changes to the pull request. 

### Further reading:

- https://www.slideshare.net/TarinGamberini/commit-messages-goodpractices
- https://gist.github.com/robertpainsi/b632364184e70900af4ab688decf6f53