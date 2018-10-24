- Start Date: 2018-10-21
- RFC PR:
- Yarn Issue:

# Summary

I want developers to have a tool to comment about packages and their corresponding version ranges.

# Motivation

My team and I found ourselves with the need to document why some packages got
pinned down older versions.

Here are the scenarios we encountered:

1. We took a decision to stay with React 15 and not to update it to React 16 and
tis lead to many packages staying on older versions as well. There's currently
no convenient way to signal the next developer who would like to
upgrade the versions of all the packages in the project, that they should keep
some on older versions and why they should.

2. Sometimes a bug is found even in minor version updates of packages and there is
no convenient way to signal developers why a package is pinned to a certain version.
I'd like a way to point to a bug and to ask the developer to unpin the package only
if it gets fixed.

3. In some cases, we cloned a repository and fixed or modified npm libraries. In
the meantime, before our pull request was merged to the root repository and uploaded
to npm, we used our forked git repository as the version of the package. There's
currently, no convenient way to document when should the git repository URL be replaced
with the newest version of a package like this (probably when the PR is merged).

4. Some packages appear in package.json to be installed in node_modules for the sake
of being used by other libraries and they are not used in the source code of the
project itself. It's very tempting to delete such packages because there's currently
no convenient way to signal the developers it's there for a reason and what's the reason.

5. Some packages can be upgraded but the new versions have breaking changes which are not easy to
cope with. In that case, it would be useful to signal a developer there's a ticket to upgrade
that specific package and what work has been done on it.

6. There's no convenient way to know what new package's versions didn't receive any
attention from the team yet.

# Detailed design

My idea is to save all this information in a separate file `yarn.comments` that
would be committed next to `package.json` and `yarn.lock`. This file would contain a
list of packages with comments on them and optionally with the pinned version the
author suggests.

This file would be easy to read and maintain both manually and automatically.

Upon running commands like `yarn upgrade` or `yarn upgrade-interactive`, comments
would appear to the user as info and warnings and confirmations about pinned versions
would run if something doesn't match. This can be suppressed with --skip-comments for
running in a CI for example. If `yarn.comments` is in use we can safely assume the
team cares about the pinned versions inside of it and can deal with adding this argument.

Upon running commands like `yarn remove` the package's comment would appear and if the
the user still wants to delete it, it would also be deleted from `yarn.comments`.

Warnings should aslo run if there is a version mismatch between `package.json` and
`yarn.comments`.

This file should be created by the user manually or as the result of choosing to do so
in one of the steps of `yarn upgrade-interactive` (Where the default is not to use it).

Comments could be added to the file as part of the *user choosing older version* for a
package in `yarn upgrade-interactive` or directly to the file, manually.

If the file doesn't exist, nothing would change for the user despite this one step in
`yarn upgrade-interactive`, which is *no* by default, with a link to the documentation
of the feature.
 
# How We Teach This

I believe adding it to the end of the documentation and adding a step in
`yarn upgrade-interactive` that would suggest the user use it with a link to the
documentation is enough.

# Drawbacks

Yarn having more files then it has now can make it seem verbose and too much "heavy",
even though using this feature is completely optional.

Having versions appear in two places can be problematic both for users and for certain
commands.

# Alternatives

I thought to use a simple "packages.md" file or adding it to the readme, but then it
gets lost and it's too easy to miss.

Another option was to somehow add it to package.json. I tried adding comments on
package.json but it's not possible. Also, I tried to add dummy packages next to packages
I wanted to comment on, but it gets messy and package managers rearrange it.

Another option would be to use "GIT" to keep track of commits where package.json got
modified, but this can also be missed and not easy to maintain.

We could also provide support for JSON5 which supports comments. But, first of all,
many libraries and tools assume the'res `package.json` and that it is in a simple
JSON format. Making them adopt to the new one is defenetly a long shot. Also-
simply comments are weaker then this suggestion in terms of best practices like
single point of truth.

Adding a block to package.json- We might support this as well or instead of using
`yarn.comments`.

We can use a library to check version missmatches vs some file or vs a block in
`package.json` upon running different `yarn` (or even npm) scripts. This looks like
the fastest solution but it doesn't feel native. I want users to see the comments when
they run `yarn upgrade-interactive` near the packages in question for example.

# Unresolved questions

I considered to suggest to use `yarn.lock` for this at first but I want the user to
be able to maintain it manually and to keep it noticeable.

Another option would be to use .yarnrc for this purpose but its purpose is different
and it's very counter-intuitive and also this file might be created automatically in
some systems.
