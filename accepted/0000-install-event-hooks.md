- Start Date: November 16, 2018
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Currently, yarn exposes `preinstall` and `postinstall` event hooks that can be defined in `package.json`. However, these event hooks are limited in that they cannot control the yarn install process. Our proposed additions are to expose event hooks in the yarn install process itself, which would enable users to short circuit the install process, if needed.

# Motivation

At the moment, `package.json` enables two life cycle hooks (`preinstall` and `postinstall`), which are useful, but limited. First, the existing hooks have no way of affecting the execution of yarn install. It would be helpful if yarn could execute scripts for various events during the installation process that could short circuit the default process. Second, users could be interested in other events than the two available currently. 

One scenario in which these hooks would be useful is to be able to run a script at the beginning of a yarn install process that can prevent yarn install from continuing. This could be very helpful for caching in CI builds where you could check an external location for a compressed `node_modules` archive that matches the current `yarn.lock`. If a matching archive is found, it could be extracted instead of running a full yarn install. In this scenario, the postinstall script could then be used to archive the `node_modules` folder and update the external cache. 

For the CI builds for large projects, this caching approach can cut a significant amount of the build time. For the purpose of this RFC, we'll use the CI builds for Visual Studio Code (VS Code) as a case study. 

Implementing such a caching strategy requires a `prefetch` and `postinstall` hook, but other events might be useful as well. For example, users may want to know when the `node_modules` folder is changed or the `yarn.lock` is updated, etc. There have been requests for these types of hooks [before](https://github.com/yarnpkg/yarn/issues/3475) so we believe it makes sense to add them at the same time.

# Detailed design

Our proposal is to add optional arguments to yarn install representing scripts that should be run for that particular hook. These arguments could also be defined in the `.yarnrc` file if the user wants to run them every time or inject them during a CI build. 

These arguments can be parsed in the `init` method of `install.js`. We would also add install steps for each hook to, if the corresponding argument was specified, run the script, otherwise skip that step. 

To enable short circuiting of the install process, the provided script could return a value indicating if the install process should continue. This value could be as simple as a `boolean` or it could be based on no `stderr` returned from the script. A specific data model would be required if we also want to support stopping the `postinstall` script of `package.json`. (Note: We believe altering the `postinstall` script of `package.json` is beyond the scope of this RFC because it could introduce breaking changes.) 

Our implementation can be seen in [this commit](https://github.com/erdennis13/yarn/commit/a3443bc47703866c13de7f5825c66ba3fba98ee1). The exposed hooks are used as optional arguments to yarn install. In the VS Code CI builds, these arguments are injected to `.yarnrc` at build time, so there would be no changes to the developer's local experience. Alternatively, the scripts could be specified as such: 

```yarn install --pre-fetch-script "node prefetchscript.js" --post-install-script "node postinstallscript.js"```

We applied the above-referenced change to yarn to the CI builds for VS Code to implement caching for the `node_modules` folder. We hash the `yarn.lock` files and use its value as the key in all of our caching. The scripts we included are defined as such: 

 - `Prefetch`. The script used to fetch a cache item first hashes a file, the `yarn.lock` file in this case, and then uses a web API layer to check if there is an existing entry for that key. If there is an entry, the script downloads the corresponding archive from an Azure Blob Storage account and unpacks it to the `node_modules` directory and short circuits the yarn install process. 
 - `Postinstall`. The script used to update the cache first creates a tarball from the `node_modules` folder. We use tar to preserve the symlinks created from yarn and to allow builds to be reproducible. This tarball is then uploaded to the web API layer and stored in the Azure Blob Storage account the `prefetch` script uses.

## VS Code build findings 

This report compares the build results from [this existing build](https://dev.azure.com/vscode/VSCode/_build/results?buildId=8882&view=logs), representing the current state of the VS Code CI builds. It compares the current state to our changes using two scenarios: when yarn installs through a [cache hit](https://dev.azure.com/1es-cat/build-cache/_build/results?buildId=199&view=logs) and when yarn installs through a [cache miss](https://dev.azure.com/1es-cat/build-cache/_build/results?buildId=186&view=logs) (followed by its normal mechanism and archive upload). 

### Cache hits vs. current state 

| Build | Current | Proposed |
|---|---|---|
| Mac  | 12:16 | 9:23 |
| Linux  | 9:56 | 7:26 |
| Windows  | 18:10 | 13:01 |

With these changes, the total build time reduced from 18 minutes to 13 minutes; representing a 27% decrease in build time. 

### Cache misses vs. current state 

| Build | Current | Proposed |
|---|---|---|
| Mac  | 12:16 | 12:50 |
| Linux  | 9:56 | 10:23 |
| Windows  | 18:10 | 19:28 | 

With these changes, the total build time increased from 18 minutes to 19.5 minutes; representing an 8% increase in build time. In analyzing the history of the VS Code repository, we found the `yarn.lock` changes in roughly 1.5% of all commits to master. This is important because it means the vast majority of builds can utilize cache hits for the package installation and benefit from the performance improvements.

# How We Teach This

This proposal won't change the way yarn works in a fundamental way and only adds functionality. However, we will certainly need to add a page to the docs detailing the various hooks and how to use them.

# Drawbacks

Our proposal increases the complexity of yarn, which could be a concern. In addition, the changes will result in additional documentation that will need to be maintained.

# Alternatives

It is possible to write scripts that wrap around yarn install without these event hooks. However, to implement the same caching functionality as we’ve done with VS Code—and which is applicable to most other projects—onboarding build scripts would require developer time and maintenance to support. Our proposal offers the caching benefits while being a drop-in solution for CI builds.

# Unresolved questions

- What hooks should be supported? 
  - At a minimum:  
    - `prefetch` (for caching, etc.) 
    - `postinstall` (for symmetry) 
  - Other ideas:  
    - When `node_modules` is updated 
    - When `yarn.lock` is updated 
- What naming convention should we follow?  
  - For example, our proposed `post-install-script` could be confused with the existing `postinstall` argument in `package.json`. 
- How do we pass parameters?  
  - We could just set environment variables, but it would be cleaner if we could pass in arguments to our script. 