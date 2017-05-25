- Start Date: 2017-05-18
- RFC PR:
- Yarn Issue: https://github.com/yarnpkg/yarn/issues/3294

# Yarn workspaces phase 3: linking workspaces to each other

## Summary

A continuation of https://github.com/yarnpkg/rfcs/pull/60.
Ability for workspaces to refer each other when testing packages in integration.

## Motivation

People tend to split larger projects into self contained packages that are published to npm independently. The workspaces feature is being developed for Yarn to address this workflow.

In particular, testing packages that refer other packages from the same codebase can be difficult because Node.js and front end bundling tools would look up the referred packages in node_modules folder as it should be installed from npm registry.

Yarn Workspaces need to be able to refer to other local packages the same way when local packages are in development mode (source of truth is the package source code) and in production mode (source of truth is the package installed from npm).

## Detailed design

The structure of the source code is following

```
| jest/
| ---- package.json
| ---- packages/
| -------- jest-matcher-utils/
| ------------ package.json
| -------- jest-diff/
| ------------ package.json
...
```

Top level package.json is like

```
{
  "private": true,
  "name": "jest",
  "devDependencies": {
  },
  "workspaces": [
    "packages/*"
  ]
}
```
jest-matcher-utils (workspace referred by another one)
```
{
  "name": "jest-matcher-utils",
  "description": "...",
  "version": "20.0.3",
  "repository": {
    "type": "git",
    "url": "https://github.com/facebook/jest.git"
  },
  "license": "...",
  "main": "...",
  "browser": "...",
  "dependencies": {
    "chalk": "^1.1.3",
    "pretty-format": "^20.0.3"
  }
}
```

jest-diff (workspace that refers jest-matcher-utils)
```
{
  "name": "jest-diff",
  "version": "20.0.3",
  "repository": {
    "type": "git",
    "url": "https://github.com/facebook/jest.git"
  },
  "license": "...",
  "main": "...",
  "browser": "...",
  "dependencies": {
    "chalk": "^1.1.3",
    "diff": "^3.2.0",
    "**jest-matcher-utils**": "^20.0.3",
    "pretty-format": "^20.0.3"
  }
}
```

When user runs yarn install, this folder structure of the Workspace gets created
```
| jest/
| ---- node_modules/
| -------- chalk/
| -------- diff/
| -------- pretty-format/
| ---- package.json
| ---- packages/
| -------- jest-matcher-utils/
| ------------ node_modules/ (empty, all dependencies hoisted to the root)
| ------------ package.json
| -------- jest-diff/
| ------------ node_modules/
| ---------------- **jest-matcher-utils**/  (symlink) -> ../jest-matcher-utils
| ------------ package.json
...
```

`jest/packages/jest-diff/node_modules/**jest-matcher-utils**` is a relative symlink to `jest/packages/jest-matcher-utils`

### Dependencies and version matching

Yarn would only link workspaces to each other if they match semver conditions.
For example,

* `jest-matcher-utils` package.json is `20.0.3`
* if `jest-diff` package.json dependencies has `jest-matcher-utils` with version specifier that matches 20.0.3, e.g. `"^20.0.3"` then Yarn will make a link from `jest-diff/node_modules/jest-matcher-utils` to `jest-matcher-utils` workspace
* if `jest-diff` package.json dependencies has `jest-matcher-utils` with version specifier that does not match `20.0.3`, e.g. `"^19.0.0"` then Yarn would fetch `jest-matcher-utils@^19.0.0` from npm registry and install it the regular way


### Problems with peer dependencies and hoisting

There is a common [peer dependency problem](http://codetunnel.io/you-can-finally-npm-link-packages-that-contain-peer-dependencies/) when using **yarn link** on local packages that people can work around in Node 6+ by setting **--preserve-symlinks** runtime flag.
In Workspaces this situation won't be a problem because node_modules are installed in Workspace root and Node.js `require()` statements will resolve third-party peer dependencies by going up the folder tree and reaching the Workspaces' root node_modules.

As long as **jest-matcher-utils** does not make relative requires via its parent folder, flag **--preserve-symlinks** won't be necessary.

### Installing workspace in project root

Workspace root may also depend on a workspace and it should be installed the same way as other workspaces referring each other, e.g. if jest has `jest-matcher-utils` as dependency it will be installed

```
| jest/
| ---- node_modules/
| -------- chalk/
| -------- diff/
| -------- pretty-format/
| -------- **jest-matcher-utils**/  (symlink) -> ../packages/jest-matcher-utils
| ---- package.json
| ---- packages/
| -------- jest-matcher-utils/
| ------------ node_modules/ (empty, all dependencies hoisted to the root)
| ------------ package.json
...
```

### Build scripts run order and cycle detection

From workspaces linking point of view installation phases look like this:

1. Resolution - Yarn identifies all workspaces and which workspaces refer each other
2. Fetching - Yarn skips it for linked workspaces
3. Linking - Yarn creates symlinks in node_modules of referring workspaces in the alphanumeric order of workspaces (starting with workspace root)
4. Running scripts - Yarn runs (pre/post)install scripts for each linked workspace the same way it runs for packages from registry. Yarn already has a way to identify cycles between packages during this phase, in this case the order of execution is not controlled by user. To control scripts execution order for cycling dependencies there is an RFC gist https://gist.github.com/thejameskyle/abbc146a8cb5c26194c8acc4d14e7c30 by @thejameskyle

## Drawbacks

This solution creates a symlink inside node_modules of a Workspace package and symlinks have multiple drawbacks:

* Symlinks are not supported in all tools (e.g. watchman)
* Symlinks are not supported well in all OS and environments (Windows pre 10 Creative updated, Docker on SMB storage(?))
* A symlink to **jest-matcher-utils** does not emulate actual installation of the package, it just symlinks to the package source code - no prepublish and postinstall lifecycle scripts are executed and no files are filtered (as done during publishing)
* A version change in package.json of **jest-matcher-utils** needs Yarn to rebuild the links, this may require file watching

## Alternatives

* Run **yarn pack** for **jest-matcher-utils** and install them from a .tgz file
    * PROS
        * Works without symlinks
        * Does not leak files from **jest-matcher-utils**, i.e. node_modules folder
        * Runs the same pack command as with real publishing to registry (tests folder and dev files won't be included)
    * CONS
        * Every file change during development of **jest-matcher-utils** will require Yarn to repack and install it  
        * Pack/unpack is an excessive use of CPU
* Hardlink files in **jest-matcher-utils** (only the ones listed for publishing) into  jest-diff/node_modules/**jest-matcher-utils**. Similar idea was expressed in the knit RFC https://github.com/yarnpkg/rfcs/pull/41
    * PROS
        * Works without symlinks' drawbacks
        * Partially emulates published package by leaving out non publishable files, e.g. node_modules folder
        * Changes in the hardlinked files will be reflected in referring workspace node_modules
    * CONS
        * Hardlinks have limited support in Windows pre 10
        * When new files are created/removed in **jest-matcher-utils** the hardlinks need to be regenerated, that may require file watching to get good developer experience otherwise developer needs to run yarn install on every significant change
        * This does not simulate actual installation of the package as no prepublish and postinstall lifecycle scripts are executed

Yarn Workspaces could implement all of the above linking strategies and give developers a choice which one to choose for their project.
Or the alternatives could be merged in a single solution for isolated e2e testing.

## Unresolved questions

* Is there an issue with Node resolving real paths in symlinked folders (https://github.com/nodejs/node/issues/3402) with this solution?
If workspaces don't make relative requires outside of their root (e.g. a file in a sibling folder to the one with workspace's package.json), all requires should resolve the same way.
* Does it need to work for other type of packages: git, file, etc?
* As described in Workspace phase 1 RFC (https://github.com/yarnpkg/rfcs/pull/60) there is only one lockfile per workspace. Does yarn.lock need to reference that `jest-matcher-utils@^20.0.0` is resolved as a link to a folder?
* Combining multiple workspaces is out of scope of this document.
