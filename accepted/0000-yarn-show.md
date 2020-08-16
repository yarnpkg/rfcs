- Start Date: (fill me in with today's date, 2020-08-16)
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Add a new command `show`, which would list the dependencies explicitly specified in `package.json`.

# Motivation

The motivation for this comes from [this issue](https://github.com/yarnpkg/yarn/issues/3569). Currently, there is no way in `yarn` to list all the explicit dependencies in a workspace.

As per the discussion in the issue, many people have mistakenly assumed that `yarn list --depth=0` would do exactly this (similar to `npm ls --depth=0`), but this is not the case.

Rather than modify the existing functionality of `yarn list` (and potentially introducing bugs and/or making it backward incompatible), it would be a good idea to introduce a new command `show` that would list just the explicit dependencies.

We could also add flags to the command to allow the user to only list dependencies of a specific type (eg. dev dependencies, peer dependencies etc.).

It would be good to show the results in a similar format to the `outdated` command.

# Detailed design

This command would be very similar to `yarn outdated`, except it wouldn't filter the results to only show the outdated packages.

We would add a new function (say, `getExplicitDependencies`) to `package-request.js`, which would be similar to the current implementation of `getOutdatedPackages`, without filtering `deps`. `getOutdatedPackages` would then be a wrapper on top of this, with the additional filter, to maintain its current functionality.

We would add a new command `show`, the implementation of which would be the same as that of `outdated`, except that it would call `getExplicitDependencies` instead of `getOutdatedPackages`.

# How We Teach This

This new command can be explained as providing information about the explicit dependencies, i.e. packages added by the user using `yarn add` in the workspace.

The corresponding NPM version would be `npm ls --depth=0`.

The Yarn documentation would have to be updated to include this new command, but no changes to the existing documentation are required. (Except perhaps a note in the documentation for `yarn list` that if the user is looking for a command to list the explicit dependencies, they might want to check out `yarn show`.)

No changes would be required in how Yarn is taught to new users at any level.

# Drawbacks

Introducing a new command to show explicit dependencies instead of modifying the existing `yarn list` command to do the same might cause confusion in users who are used to NPM, since NPM already handles this as part of `npm ls` (which is analogous to `yarn list`).

# Alternatives

This is completely new functionality, with no current workaround.

The other design choice would be to modify `yarn list` to show explicit dependencies only (perhaps using a new flag).

# Unresolved questions

* Is there a way to avoid duplication of code when implementing `show` (which is extremely similar to `outdated`)?