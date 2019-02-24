*   Start Date: 2017-03-18
*   RFC PR: (leave this empty)
*   Yarn Issue: (leave this empty)

# Summary

This RFC proposes the addition of a "upgrade-transitive" command to the Yarn CLI.

This will provide developers a command to add dependencies to the `package.resolutions` field, to override the versions of transitive dependencies installed.

# Motivation

It is occasionally necessary to upgrade versions of transitive dependencies in an application (in the case of security patches or critical bug fixes) without going through the regular workflow. Currently, to upgrade transitive dependencies, a developer would have to hand edit the yarn.lock file (which is an anti-pattern; reasons as to why this must not be done are outside the scope of this RFC).

Consider the dependency graph A->B->C. (A is an application that lists B in its `package.json`; B lists C in its `package.json`.) The regular workflow would be to bump the minimum required version of C in B, and bump the new minimum required version of B in A.

This causes friction however, when B is not already at the latest version in A, and is sufficiently large that upgrading to a new version of B in A is non-trivial.

By allowing the upgrading of transitive dependencies listed in the lockfile, bug fixes and security patches can be rolled out quicker.

# Detailed design

## Using the `resolutions` field

Yarn already has a mechanism for overriding which versions of transitive dependencies to install. The [selective dependency resolutions](https://yarnpkg.com/lang/en/docs/selective-version-resolutions/#toc-why-would-you-want-to-do-this) field in `package.json` provides the API for this. We will use this API in our implementation for the following reasons:

*   The logic to override which packages yarn installs already exists here. By using this, we minimize the amount of duplicated logic.
*   Information about which packages have been overridden will live in `package.json`, and not just `yarn.lock`. This is preferable since `yarn.lock` may not exist in all projects, or [may be regenerated](https://github.com/yarnpkg/yarn/pull/3544).

The new command should be very vocal about making changes to `package.resolutions` and inform users of this, as this is ideally a change that package authors should want to revert as soon as possible.

## A new CLI option

We will introduce a new command to Yarn; `yarn upgrade-transitive`. (This naming matches the already existing `upgrade-interactive`).

The command will accept a space-separated list of arguments, where each argument corresponds to a package path and a version range, where the package path and version range are valid, according to the [selective dependency resolutions](https://yarnpkg.com/lang/en/docs/selective-version-resolutions) spec.

### Examples:

```
$ yarn upgrade-transitive d2/left-pad@1.1.1
$ yarn upgrade-transitive d2/left-pad@1.1.1 c/**/left-pad@1.1.2
```

When invoked, the following actions are taken:

*   Write the change to the `resolutions` field in `package.json`
*   Trigger behaviour equivalent to a regular `$ yarn` call
    *   (i.e. Update `node_modules` and the lockfile if present)

_([Prior art for triggering an install](https://github.com/yarnpkg/yarn/blob/a35a4f97dd7e5f6f029b5e0f855bfd73ef338ca4/src/cli/commands/upgrade.js#L179) - the `$ yarn update` command)_

## Stretch Implementation Goal

In some cases, the new version of the transitive dependency may be in range as specified by its parent. (Or a newer in-range version of the parent).

This would mean that an override in `package.resolutions` would not be required, and instead can be installed if the versions of the parent and its dependencies were re-resolved.

The second milestone for the implementation of this command should first attempt to see if this can be achieved, and do so before falling back to using `package.resolutions`.

# How We Teach This

The Yarn CLI should add a `upgrade-transitive` to the output of `--help` commands, explaining that this will allow developers to pass in the name/path to a dependency that should be updated.

Yarn should use the term "transitive" to describe the sub-dependencies as this is a mathematically accurate description, and more precise than "sub-dependencies" or "additional dependencies". (npm also uses this term (http://blog.npmjs.org/post/145724408060/dealing-with-problematic-dependencies-in-a).

The new feature:

*   Is an addition of functionality. The documentation will need amending to describe the new command.
*   Is backwards compatible, does not change other existing functionality of Yarn, and would not require existing users to change their workflow in any way.
*   Should be taught via release notes and a blog post at time of release. The documentation and `--help` output will serve as a reference for future users.

# Drawbacks

It is preferred to handle upgrades of transitive dependencies via the "regular workflow" as described in the motivation section above. Introducing this as an additional workflow could confuse or engender bad practices. The documentation of the flag should be written with care to reduce this as a possibility.

In addition, since the implementation uses the [selective dependency resolutions](https://yarnpkg.com/lang/en/docs/selective-version-resolutions) API, we should be mindful of the "Limitations & Caveats" stated:

> *   Nested packages may not work properly.
> *   Certain edge-cases may not work properly since this is a fairly new feature.

# Alternatives

## Existing workarounds

If the version of the desired transitive dependency is within range as declared by its parent, one can manually trigger yarn to re-resolve this information by re-adding the parent dependency through existing CLI APIs. This may also lead to hand-editing of lockfiles.

If the version of the desired transitive dependency is _not_ within range as declared by its parent, users can wait for the parent to release a new version which includes the new transitive dependency.
This release cycle may slow down the release of critical fixes.

## Manually editing the `resolutions` field

Users could manually edit `package.json` to target the new version of the desired transitive dependency, without going through the CLI.

https://yarnpkg.com/lang/en/docs/selective-version-resolutions/

# Other Suggested APIs

*   As proposed in https://github.com/yarnpkg/yarn/issues/2394, `$ yarn upgrade --deep`

# Unresolved questions

*   Is there a possibility of introducing transitive dependency version conflicts that cannot be resolved?
*   What happens when a new version of a transitive dependency introduces a new dependency?
