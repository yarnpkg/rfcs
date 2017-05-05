- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Workspaces adds support for managing multiple packages within a single Yarn
project. Linking between them on install to make cross-development simpler.

# Motivation

It can be difficult to develop across packages. Especially when trying to test
changes across many different packages.

Additionally, the cost of abstracting code into it's own package is too high of
an additional maintenance cost. So authors will often avoid abstracting major
pieces of tools into their own packages because it would make development
harder.

If Yarn had a way of developing many packages as a single project which removed
the additional maintenance cost of being able to test changes across packages,
it would encourage more tools to abstract core functionality out.

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with Yarn to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## Configuration

The top-level project `package.json` may specify a `"workspaces"` field which
contains an array of file path globs (relative to the directory of the
project's `package.json`) which point to directories where a workspace
`package.json` can be found.

```json
{
  "name": "my-project",
  "workspaces": [
    "package-one",
    "package-two",
    "packages/*"
  ]
}
```

Workspace `package.json`'s do not have any additional configuration from a
standard package.

***[WIP]***

## Commands

### Filtering

Flags which can be added to commands to allow them to filter to a subset of
packages (including the project package and workspace packages).

- `--only [glob]` - Whitelist workspace package names (not directories)
- `--ignore [glob]` - Blacklist workspace package names (not directories)
- `--only-fs [glob]` - Whitelist workspace package directories
- `--ignore-fs [glob]` - Black list workspace package directories

**Examples:**

```sh
yarn [command] --only package-name-{a,b}
yarn [command] --ignore @scope/util-name-*
yarn [command] --only-fs packages-dir/{a, b}
yarn [command] --ignore-fs utils-dir/*
```

### Global commands

Each of these need to be modified to consider

- `yarn check`
- `yarn clean`
- `yarn install`
- `yarn pack` ([filterable](#filtering))
- `yarn publish` ([filterable](#filtering))
- `yarn version` ([filterable](#filtering))
- `yarn why`

### Workspace commands

Commands for running sub-commands within workspaces.

- `yarn workspaces [command]` / `yarn ws` ([filterable](#filtering))
- `yarn workspace [workspace] [command]` / `yarn w`

##### Allowed sub-commands

- `add` / `upgrade` / `remove`
- `link` / `unlink`
- `run` (including aliases: `test`, `start`, etc.)
- `tag` / `owner`

### Scripts

When running a script like `yarn test` or its expanded form `yarn run test`, it
should behave like this:

1. Look for `"test"` script in root project.
  - If it exists, run that script. Exit with the result.
2. Look up `"test"` script in each individual workspace.
  - If one or more exist, run them (in parallel?). Exit with the results.
3. If no test scripts exist, exit with an error.

## Install Process

With workspaces, installing will be changed to manage the dependencies of every
package within a project.

As much as possible Yarn should try to treat the set of dependencies across
workspaces as a single set. This includes having a single `yarn.lock` file and
resolving the dependency tree as a whole. The exception being where
dependencies get placed. If `workspace-a` depends on `dep@1-2` and
`workspace-b` depends on `dep@1-3`, it will resolve to a single `dep@2` but
will place copies of `dep@2` in both workspaces' `node_modules`.

This avoids adding complexity to the Yarn codebase which would have to manage
multiple trees of dependencies, resolving them separately. This ends up being a
better behavior for most users anyways. It also (likely) ends up being much
faster since it does not have to resolve dependencies for each workspace.

#### `yarn.lock`

Treating the install process of an entire project including its workspaces as a
single dependency tree means Yarn can have a single `yarn.lock` file and will
not have to modify it at all.

#### `--link-duplicates`

The `--link-duplicates` flag should work the same exact way it does today,
except it can link across workspaces' `node_modules`.

This could get a bit weird trying to find the actual contents of a dependency
in your file system since they could be in any workspace. I'm also proposing a
`/node_modules/.hoisted` directory, and I'd recommend `--link-duplicates` be
modified to use that all the time.

#### `--flat`

Since dependencies of all workspaces get resolved at the same time, resolutions
should be stored within the project's `package.json`.

We need to find a good way of displaying which packages depend on which version
ranges. If you list all of them at once it could get massive.

## Hoisting

This is a separate RFC but affects workspaces. Note that it only works when you
are using `--flat` (because you can't create `node_modules` within symlinks),
and using `--hoist` should probably imply `--flat`.

Alternatively it could only hoist dependencies that don't require nesting.

#### `--hoist`

The goal here is to have a reliable location for every dependency to live
inside which is a flat structure that gets symlinked into every package's
`node_modules`.

#### `/node_modules/.hoisted`

Dependencies should be placed within the project's `node_modules/.hoisted`
directory with

```
/node_modules/
  /.hoisted/
    /dependency-a-v1.0.0/(contents)
    /dependency-a-v2.0.0/(contents)
    /dependency-b-v1.0.0/(contents)
    /dependency-c-v3.0.0/(contents)
  dependency-a -> ./.hoisted/dependency-a-v1.0.0/
  dependency-b -> ./.hoisted/dependency-b-v1.0.0/
```

All of the `node_modules` within workspaces should also link back to the
project's `node_modules/.hoisted` directory.

```
/packages/
  /workspace-a/node_modules/
    dependency-a -> ../../../node_modules/.hoisted/dependency-v2.0.0
    dependency-c -> ../../../node_modules/.hoisted/dependency-v3.0.0
  /workspace-b/node_modules/
    dependency-a -> ../../../node_modules/.hoisted/dependency-v1.0.0
    dependency-b -> ../../../node_modules/.hoisted/dependency-v1.0.0
```

## Versioning

When using workspaces, `yarn version` should not allow `major/minor/patch/etc`.
It should error and tell you to use just `yarn version`. Using that should
iterate through each workspace and prompt for a new version.

Right now we use a `question` field to manually type in a version. However, we
should change that to a multi-choice selector
(See [Inquirer.js](https://github.com/SBoudrias/Inquirer.js/#list---type-list)).

```sh
info Package: workspace-a
info Current version: 1.0.0
question New version:
  Skip
  Patch (1.0.1)
> Minor (1.1.0)
  Major (2.0.0)
  Custom
```

As you go through the items, the selector should default to the previously
selected version choice. So if you picked "patch" previously, the next prompt
would default to "patch".

### With Git

Should look at the git diff since the last tag and see if there were any
changes to each workspace.]

If there were no changes, default the version selector to "Skip" which does not
create a version.

```sh
info Package: workspace-a
info Current version: 1.0.0
question New version:
  Diff (no changes)
> Skip
  Patch (1.0.1)
  Minor (1.1.0)
  Major (2.0.0)
  Custom
```

There should also be a diff option to open up a scroller to view the diff, when
you exit it brings you back to the version selector.

```sh
info Package: workspace-a
info Current version: 1.0.0
question New version:
> Diff (changes: +46, -14)
  Skip
  Patch (1.0.1)
  Minor (1.1.0)
  Major (2.0.0)
  Custom
```

##### `--skip-unchanged`

If you want to automatically skip packages that have diff since their last tag
you can run `yarn version --skip-unchanged` to do so.

### Without Git

Should go through every workspace and prompt you for a new version with the
option not to create a new version.

### On failure

If for any reason creating a new tag fails, we should roll everything back
immediately.

## Publishing

### Temporary tags

If you publish everything as latest immediately you end up causing builds to
break while it's running (npm publishing lots of packages takes a long time).

Instead you need to publish all the packages to a temporary tag on npm and then
move them over to "latest" all at once.

For example:

- Publish `dependency-a@1.0.1#yarn-temp`
- Publish `dependency-b@1.1.0#yarn-temp`
- Publish `dependency-c@2.0.0#yarn-temp`
- Tag `dependency-a@1.0.1` `latest`
- Tag `dependency-b@1.1.0` `latest`
- Tag `dependency-c@2.0.0` `latest`

Updating tags is much faster than publishing so this ends up breaking less.

### Deciding which packages to publish

Since `yarn version` handles creating new versions, we don't know which
workspaces need publishing.

We could just lookup the current version of every package on the registry but
that leads to race conditions if someone else tries to publish at the same
time (a rarity, but could easily happen within larger organizations).

Instead, we first go through every package and "lock" them by publishing a
`yarn-lock-{unique id}` tag to each package's highest version.

If we discover an existing `yarn-lock-{unique id}` tag in this process we roll
back the tags immediately and tell the user we think someone else is
publishing right now because of the `yarn-lock-{unique id}` tag we discovered.

Once we have everything locked we go through and lookup the latest version of
every package.

If we have new versions locally, we queue those up to be published.

After publishing (including on failure) we roll back the `yarn-lock-{unique id}`
tags.

### On failure

If a single package fails to publish, we should continue publishing the rest.
Which might seem unintuitive, but oftentimes the author will just have to
create a new version for just that package and run publish again to make it
work.

The alternative is to leave a bunch of packages in half finished states
which means the author has to go through and fix them all individually.

## Scripting

Yarn should include a number of utility commands to make scripting easier.

### `yarn exec` / `yarn ws exec`

`yarn exec` is a new command which executes another command (separated by `--`)
with `/node_modules/.bin` in the `$PATH`.

```sh
$ which babel
# does not exist
$ yarn exec -- which babel
/Users/me/code/my-project/node_modules/.bin/babel
```

If you want to run a command within every single workspace, you can do that via
`yarn workspace exec` (or `yarn ws exec`) like so:

```sh
$ yarn workspaces exec -- pwd
/Users/me/code/my-project/packages/package-a
/Users/me/code/my-project/packages/package-b
/Users/me/code/my-project/packages/package-c
```

There's three options for what the `$PATH` should be within workspaces.

- The project's `node_modules/.bin`
- The workspace's `node_modules/.bin`
- Both (w/ workspace before project)

## Differences from Lerna

1. No "fixed" mode - Keeping every package at the same version will require
  manually doing so. This is a source of complexity in the Lerna codebase which
  is not necessary.
2. Resolves all packages at once and shares version constraints across
  workspaces
3. Does not integrate with git as tightly

# How We Teach This

### Terminology

We'll have three terms when describing a codebase using Yarn:

- **Package**: A directory which contains a `package.json` and all of its code.
- **Project**: A top-level _package_ (generally a repository) which might
  specify nested _workspaces_.
- **Workspace**: A package which which is specified by the project nested
  within the project. The top-level project package may also be specified as a
  workspace.

All packages are installable and publishable, including the project package and
any workspace packages.

Right now we only have project packages, the Yarn docs are already using the
term "project" to describe them this way.

### Documentation changes:

- New guide teaching what workspaces are including how to create and use them.
- Commands that have additional behavior around workspaces will need to be
  updated.
- Additional commands will have to be documented.

# Drawbacks

- It could add a lot of complexity to the codebase.
- Adding additional languages in the future will have to solve problems around
  linking as well.

# Alternatives

- Develop as a separate tool (like [Lerna](https://lernajs.io/)). Which might
  be good because it would force us to have a detailed programatic api, but
  also might be bad because it would expose a lot of internal behavior of Yarn.

# Unresolved questions

1. What should be the behavior of running commands in nested workspace
  packages? Should they treat that as the top-level package or lookup to see
2. Where should `yarn workspace[s] pack` place all of the `.tgz` files
  (top-level or inside each workspace)?

***[WIP]***
