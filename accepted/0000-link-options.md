- Start Date: 2017-01-13
- RFC PR: https://github.com/yarnpkg/rfcs/pull/41
- Yarn Issue: https://github.com/yarnpkg/yarn/issues/1213

# Summary

This is a proposal to improve upon `yarn link` so that developers can more accurately test in-development versions of their libraries from their apps or other libraries.

# Motivation

`yarn link` (and `npm link` before it) have several problems when working on code bases of non-trivial sizes, especially with multiple apps. The current `link` command doesn't isolate `node_modules` between apps (especially problematic with the advent of Electron), it doesn't allow for working on multiple versions of a library, and it produces a `node_modules` hierarchy that is not faithful to the one produced after the library is published.

# Detailed design

## Desired behavior

The `yarn link` workflow should mimic publishing a package (ex: `dep`) to npm and then installing it in a dependent (ex: `app`), and keep this constraint while you're making changes to the first package. Concretely, `yarn link` should make it so that when you save a change to `dep`, the resulting state is as if you:
1. Ran `npm publish` in `dep` (assume that it can clobber an existing version, and that you're publishing to a local registry on just your computer).
2. Ran `yarn add dep` in `app`.

## Why this behavior is great

This solves several problems that "yarn link" has today:

#### Isolating `node_modules` correctly

You can install `dep` in two different apps without sharing the `node_modules` of `dep`. This is a problem with Electron apps, whose V8 version is different than Node's and uses a different ABI. If you have `node-app` and `electron-app` that both depend on `dep`, the native dependencies of `dep` need to be recompiled separately for each app; `node-app/n_m/dep/n_m` must not be the same as `electron-app/n_m/dep/n_m`.

#### Working on multiple versions

You can be developing multiple different versions of `dep`. Say you have two directories, `dep-1` and `dep-2`, which have your v1 and v2 branches checked out, respectively. With "yarn link" it's not possible to make both of these directories linkable at the same time.

This is a problem when you are developing & testing `dep-1` with `old-app` and `dep-2` with `new-app`. You don't want to be going back and forth between `dep-1` and `dep-2` running "yarn link" each time you switch which app you're testing.

#### Faithfully representing the `node_modules` hierarchy

Currently `yarn link` symlinks the entire package directory, which brings along its `node_modules` subdirectory with it. With dependency deduping and flattening, bringing in `dep/node_modules` wholesale usually produces a different `node_modules` hierarchy than running `yarn install` in `app` and installing everything from npm. This isn't a problem most of the time but it does go against Yarn's spirit of consistency and the lockfile.

## A practical proposal

This is a proposal that solves all of the problems above and isn't too hard to implement or understand. Conceptually, we find all the files we'd normally publish to npm, pack them up using symlinks instead of copies of the files, publish the pack to a local registry (just a directory), and then when installing we look up packages in the local registry directory instead of npm.

### Running "yarn link --pack" inside of `dep`

This is the step that simulates publishing `dep`. Running `yarn link --pack` in `dep` finds all the files that "yarn publish" would pack up and upload to npm. Crucially, this excludes `node_modules`, and would follow the same algorithm as "yarn publish" such as reading package.json's `files` field.

Then it simulates publishing `dep`: it creates a directory named `dep-X.Y.Z` (where `X.Y.Z` is the version of `dep` in its package.json) inside of a global directory like `~/.yarn-link`. A symlink is created for each file or directory that `yarn publish` would normally have packed up. This step shares some conceptual similarities with publishing to a registry, except it uses symlinks and it's local on your computer.

### Running "yarn add dep --link" inside of `app`

This behaves like normal `yarn add dep` except that it looks at the versions of `dep` that are in the global `~/.yarn-link` folder and takes the latest one rather than installing from a normal registry. (You also could run "yarn add dep@X.Y.Z --link" if you wanted a more specific version, as usual.)

It then runs most of the same installation steps that `yarn add dep` would. It symlinks `app/node_modules/dep` to `~/.yarn-link/dep-X.Y.Z` as `yarn link dep` does. Then it installs the dependencies of `dep` as usual by fetching them from npm. Finally it runs postinstall scripts.

### Running "yarn install --link" inside of `app`

This behaves like normal `yarn install` but automatically links any dependencies that have been set up locally with `yarn link`. This is very useful when you have an application with many locally developed dependencies so that you don't have to manually run `yarn link dep` for each one inside your app.

# Comparison with other Yarn features

There are already several ways of linking dependencies in Yarn. This section is a comparison of them, and explains why these new options are still valuable to add.

* `yarn link` without the `--pack` option is similar to `yarn link --pack` but links directories instead of files. This means that any `node_modules` inside linked dependencies are also included, breaking the flattened and deduped tree that yarn normally provides. Many other differences have already been discussed in this RFC. Ideally, the behavior of `yarn link` would be what the `--pack` option provides, but for backwards compatibility and to match the behavior of `npm link`, `yarn link` would stay around as is.

* `link://` dependencies are similar to `yarn link dep` but actually closer to what `yarn add dep --link` would provide. `link://` dependencies also cause the linked module's dependencies to be installed locally, so as long as there is no `node_modules` folder inside the linked dependency, it would work identically to `yarn add dep --link`. The existing implementation of `link://` dependencies could be used to implement the `--link` option.

* `file://` dependencies cause files to be copied instead of linked. This is not as useful because it means you must re-install every time a change is made to the dependency.

* `yarn pack` + `yarn install dep.tgz` is similar to `file://` dependencies. The pack + install process must be re-run every time a change is made. It does correctly dedupe dependencies, however, as `node_modules` are excluded by `yarn pack`.

* [Workspaces](https://github.com/yarnpkg/rfcs/pull/66) are similar but solve a different problem. Where workspaces are great for a tree of related modules (e.g. a monorepo), these options are for linking together modules in separate trees, e.g. things that might be shared between multiple workspaces.

# How We Teach This

This proposal is mostly additive and affects only how people work on libraries that they are using in their apps. We would want to document the new options in the "CLI Commands" section of the docs and perhaps add a new section to "The Yarn Workflow".

`yarn link` would stay around, so people migrating from the npm client wouldn't have to learn anything new at first.

# Drawbacks

One issue with this proposal is that it's not clear what to put in the lockfile after running `yarn add dep --link` since we don't have an npm URL for the dep yet -- it hasn't been published to npm.

Another issue is that if you change package.json in `dep`, namely changing a dependency or modifying the `files` entry, you have to run `cd dep; yarn link --pack`. Same if you add a file at the top level of the dependency since each file is linked individually. This isn't so bad as those changes are probably more rare than saving changes to existing files.

Also, if you update the code in dep and bump its version, say from 1.0.0 to 1.1.0, the symlinks in ~/.yarn-link/dep-1.0.0 will still point to the code in your working directory, which now contains 1.1.0 code.

The symlinks might break but I think that's mostly OK since at that point you're done working on dep and have published it to npm and it's easy to go run `yarn add dep` in app and not use the symlinks anymore.

If you want to truly pin the versions of linked packages then you'd need to have a different working directory for each version. (Git worktrees are great for this use case actually. Worktrees let you check out a repo once and then magically create semi-clones of it in separate directories, with the constraint that the worktrees need to be on different branches, which is totally OK in this scenario. The worktrees all share the same Git repo though, so if you commit in one worktree you can cherry pick that commit within another worktree.)

Finally, another issue is with the way the node require resolution algorithm works. Dependencies of symlinked modules are resolved relative to the realpath, not the symlink. This means that you'll still get duplicates if both modules depend on a third dependency, or errors if that dependency is not installed in either place. This is solved by the recently added runtime option for node `--preserve-symlinks`, which skips getting the realpath when resolving modules. Something similar would need to be added to browserify/webpack to solve this there as well. I recently opened a [PR for browserify](https://github.com/substack/node-browserify/pull/1742) to support the same option.

# Unresolved questions
