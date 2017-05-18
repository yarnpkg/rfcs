- Start Date: 2017-05-04
- RFC PR:
- Yarn Issue: https://github.com/yarnpkg/yarn/issues/3294

# Summary

Add workspaces keyword to package.json to allow multi project dependency installation and management.
When running install command yarn will aggregate all the dependencies in package.json files listed in workspaces field, generate a single yarn.lock and install them in a single node_modules.

# Motivation

This RFC is based on a need to manage multiple projects in one repository [issue #3294 aggregates all related discussions](https://github.com/yarnpkg/yarn/issues/3294 (https://github.com/yarnpkg/yarn/issues/884)).

[Image: https://fb.quip.com/-/blob/CTYAAAX6p4Y/3y0pmJnH2NKkg4ae06hVyw]
We are taking an iterative approach to implement the whole end to end experience of workspaces, there will be RFC and PRs for:


1. Installing dependencies of all workspaces in the root node_modules (this RFC)
2. Commands to manage (add/remove/upgrade) dependencies in workspaces from the parent directory
3. Ability for workspaces to refer each other, e.g. jest-diff → jest-matcher-utils
4. package-hoister for workspaces, e.g. when dependencies conflict and can't be installed on root level
5. Command to publish all workspaces in one go

For a while workspaces will be considered an experimental feature and we will expect breaking changes until it becomes stable.

# Detailed design

I'll use [Jest](https://github.com/facebook/jest) for the example implementation.

Workspaces can be enabled by a flag in .yarnrc:
```
yarn-offline-mirror "path"
disable-self-update-check true
workspaces-experimental true
```

The structure of the source code is following

```
| jest/
| ---- package.json
| ---- packages/
| -------- babel-jest/
| ------------ packjage.json
| -------- babel-preset-jest/
| ------------ packjage.json
...
```

Top level package.json is like

```
{
  "private": true,
  "name": "jest",
  "devDependencies": {
    "ansi-regex": "^2.0.0",
    "babel-core": "^6.23.1,
  },
  "workspaces": [
    "packages/*"
  ]
}
```
babel-jest
```
{
  "name": "babel-jest",
  "description": "Jest plugin to use babel for transformation.",
  "version": "19.0.0",
  "repository": {
    "type": "git",
    "url": "https://github.com/facebook/jest.git"
  },
  "license": "BSD-3-Clause",
  "main": "build/index.js",
  "dependencies": {
    "babel-core": "^6.0.0",
    "babel-plugin-istanbul": "^4.0.0",
    "babel-preset-jest": "^19.0.0"
  }
}
```

babel-preset-jest
```
{
  "name": "babel-preset-jest",
  "version": "19.0.0",
  "repository": {
    "type": "git",
    "url": "https://github.com/facebook/jest.git"
  },
  "license": "BSD-3-Clause",
  "main": "index.js",
  "dependencies": {
    "babel-plugin-jest-hoist": "^19.0.0"
  }
}
```

If workspaces is enabled and yarn install is run at the root level of jest Yarn would install dependencies as if the package.json contained all the dependencies of all the package.json files combined, i.e.

```
{
  "devDependencies": {
    "ansi-regex": "^2.0.0",
    "babel-core": "^6.23.1,
  },
  "dependencies": {
    "babel-core": "^6.0.0",
    "babel-plugin-istanbul": "^4.0.0",
    "babel-preset-jest": "^19.0.0"
    "babel-plugin-jest-hoist": "^19.0.0"
  }
}
```

## Resolving conflicts

The algorithm is the same as in Yarn's [hoisting algorithm](https://github.com/yarnpkg/yarn/blob/master/src/package-hoister.js) during linking phase.

In the example above babel-core is used in both top level package.json and one of the workspaces.
Yarn will resolve the highest possible common version and install it.
If versions are conflicting Yarn will install install the most common used one at the root level and install the other versions in each of workspaces folder.

This should be enough for Node.js to resolve required dependencies when running in each of the workspaces.

### Note: In the first implementation workspaces level hoisting won't be implemented and Yarn would throw an error in case of dependency conflicts between packages.

### Note: linking, i.e. workspaces referring each other is not covered in this RFC, it will come in a next phase

## yarn.lock

After running yarn install at the top level Yarn will generate a yarn.lock for all the used dependencies at the root level in workspaces and save it only at the root level.
Yarn won't save yarn.lock files at workspaces' folders.


## Running yarn install in workspaces folders

Yarn will automatically run all commands as if running in the root folder, i.e. install won't install node_modules in workspaces' folders individually.

## Check command and integrity check

Changes will be needed to [check command](https://github.com/yarnpkg/yarn/blob/master/src/cli/commands/check.js) and [integrity-checker](https://github.com/yarnpkg/yarn/blob/master/src/integrity-checker.js) considering the new patterns added during resolve phase.


# Drawbacks

Not sure how exactly cross-platform symlinks are today. However it looks like
[Microsoft might be catching up](https://blogs.windows.com/buildingapps/2016/12/02/symlinks-windows-10/) on this issue.

# Alternatives

Lerna as used in [jest](https://github.com/facebook/jest) now.
Having multi-project dependency management natively in Yarn gives us a more cohesive user experience and because Yarn has access to dependency resolution graph the whole solution should provide more features than a wrapper like Lerna.

# Unresolved questions

Running lifecycle scripts may cause unexpected results if they require a specific folder structure in node_modules.
