- Start Date: 2017-06-21
- RFC PR: n/a
- Yarn Issue: https://github.com/yarnpkg/yarn/issues/3603

# Summary

Spawned from https://github.com/yarnpkg/yarn/issues/3603

There is a lot of confusion among new users of Yarn as to how the `upgrade` and `upgrade-interactive` commands work.

A lot of that confusion is due to those commands not working the same way (nor are they implemented the same).

The purpose of this RFC is to align those commands, and begin to share implementation between them.

# Motivation

## Currently (yarn <=0.26):

`upgrade` = upgrade all packages to their latest version, **respecting** the range in `package.json`
`upgrade left-pad` = upgrade only the left-pad package to it's `latest` tag, **ignoring** the range in `package.json`
`upgrade-interactive` = upgrade all packages to their `latest` tag, **ignoring** the range in `package.json`

It is very confusing that `upgrade` vs `upgrade-interactive` chose different versions, and `upgrade` vs `upgrade {package}` chose different versions.

# Detailed design

Major design ideas:

1. `upgrade-interactive` should just be an "interactive" version of `upgrade`.
2. Both commands should respect the package.json semver range by default.
2. PR #3510 added a `--latest` flag to `upgrade` to tell it to ignore package.json range. Utilize this change across both commands to have them ignore package.json range and use `latest` tag instead.

## New Logic:

* Leave upgrade with no additional parameters how it is:
> yarn upgrade
>
> This command updates all dependencies to their latest version based on the version range specified in the package.json file.

* Change passing a package without an explicit version to respect package.json
> yarn upgrade [package]
>
> This upgrades a single named package to the latest version based on the version range specified in the package.json file.

* Leave handling an explicit version the same as how it is
> yarn upgrade [package@version]
>
> This will upgrade (or downgrade) an installed package to the specified version. You can use any SemVer version number or range.

* Utilize the --latest flag from PR #3510 for an upgrade without a specific package, and add it to the docs
> yarn upgrade --latest
>
> This command updates all dependencies to the version specified by the latest tag (potentially upgrading the package across major versions).

* Utilize the --latest flag from PR #3510 for an upgrade with a specific package, and add it to the docs
> yarn upgrade [package] --latest
>
> This upgrades a single named package to the version specified by the latest tag (potentially upgrading the package across major versions).

For `upgrade-interactive` it would internally just call the `upgrade` logic to follow the same rules above, but would then present the list of packages to the user for them to chose which to upgrade. The exception is that `upgrade-interactive` does not have the ability to take specific package names in its parameters (because the user would chose them from the interactive selection list instead of specifying them on the cmd line)


## Implementation Details

Currently, `upgrade` reads all packages and ranges from package.json and forwards them to `add`. `upgrade-interactive` is implemented differently; it uses `PackageRequest.getOutdatedPackages()` to determine only the packages that are out of date, and what version they would update to.

As part of this work, the upgrade-interactive logic to use `getOutdatedPackages` would be moved over to `upgrade`.

`PackageRequest.getOutdatedPackages()` already reports the "wanted" (latest respecting package.json specified range) and the "latest" (latest specified by registry, ignoring package.json) versions for all outdated packages. `upgrade` would look for the `--latest` flag to decide which of these version to upgrade each package to.

The `upgrade-interactive` command's output will include an additional column named "range". This column will show what the current package.json specified range is. If the `--latest` flag is passed, then the word "latest" will be displayed. In other words, this column is showing the range specifier that upgrade is using to determine what to upgrade to.

example:

```
 dependencies
   name        range   from        to      url
❯◯ chai        ^3.0.0  3.4.0    ❯  3.5.8   http://chaijs.com
```

Which indicates "You have chai@^3.0.0 as a dependency. Currently 3.4.0 is installed. Upgrade will move to 3.5.8"

or when using the --latest flag:

```
 dependencies
   name        range   from        to      url
❯◯ chai        latest  3.4.0    ❯  4.0.2   http://chaijs.com
```

The goal here is to better explain to the user why this version was selected to upgrade to.


## Preserve package.json range operator

Related to #2367 and #3609 there have been requests that `upgrade` and `upgrade-interactive` when upgrading to a new major version will preserve the `package.json` specified version range, if it exists.

So for example if package.json specifies the dependency

```
  "foo": "~0.1.2",
  "bar": "^1.2.3",
  "baz": "2.3.4"
```

Then if `upgrade --latest` jumps to a new major version, it will preserve the range specifiers, and upgrade to something like:

```
  "foo": "~5.0.0",
  "bar": "^6.0.0",
  "baz": "7.0.0"
```

(with the current implementation, all 3 packags would be changed to use caret `@^x.x.x`)

* This only has an affect when `--latest` is specified, otherwise package.json file would not be modified.

* This only works for simple range operators (exact, ^, ~, =, <=, >). Complex operators are not handled. When a range operator is not one of these simple cases, `^` will be used as the default., since that is the normal range operator when adding a package the first time.

* This behavior is overriden by the following flags: `--caret` `--tilde` `--exact`. If any of these are passed, then that range operator will always be used.


# How We Teach This

Docs will need to be updated to reflect these changes.


# Drawbacks

This is a change to the version ranges selected by `upgrade-interactive` so could cause additional confusion to those who use it in previous Yarn versions.

However, I beleive that overall it would reduce the confusion between these commands, and make them overall more versitile.


# Alternatives

Do not change the behavior. Deal with user confusion and issues that arrise from it as they come up.


# Unresolved questions

None at this time.
