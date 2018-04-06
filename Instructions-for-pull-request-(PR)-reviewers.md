What to check
-------------

- [ ] TODO

How to merge
------------

Ginkgo has two merge methods enabled: "_merge_" and "_squash and merge_". You should use the appropriate method depending on how the history of the pull request looks like:

*   If the history is nice, clean and linear, use _merge_. This will preserve the entire history for future reference.
*   If there's something crazy going on (50 commits for 3 lines of changes, merge commits within the feature branch, commits that remove changes introduced by other commits, etc.), just use "_squash and merge_". This will destroy the history of the pull request, but will keep the history of Ginkgo clean.

Make sure you __edit the commit message__ of the merge request. The default Github's message "Merge pull request #2982 from project-name/branch-name" is not very useful. There's no other way to get a commit in `develop`/`master` other than by merging a pull request and the PR number doesn't tell you much if you don't have access to github later. In addition, both Github and Gitlab (where the private fork of Ginkgo is) use hashes to denote issues, so using them in commit messages adds confusing back-references to issue trackers. Instead, you should describe in short what you are merging (e.g. "Merge CG reference kernels"), and add a __full link__ to the related pull request in the description section of the commit message.