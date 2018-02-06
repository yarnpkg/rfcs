* Start Date: 2017-09-21
* RFC PR:
* Yarn Issue:

# Summary

Allow Yarn CLI to execute a package script on each workspace package from the workspace root.

# Motivation

Lerna does a great job for handling monorepos. Yarn's built-in workspaces feature would benefit from borrowing more of functionality that Lerna exposes, to handle monorepos more easily.

Just like installing all dependencies from one place (workspace root), it'd be useful to execute a specific package script for each of the workspace packages from the workspace root using a single command. Not only this would make for less back-and-forth traveling between the packages to execute scripts, but also it'll help a lot for CI/CD configuration.

# Detailed design

As @BYK mentioned [here](https://github.com/yarnpkg/yarn/issues/4467#issuecomment-330873337), to start we'd create two commands for this feature:

* `yarn workspaces list`
* `yarn workspaces run <command>`

Unifying all of the workspace-specific commands under the `workpaces` namespace allows for a modular approach from the CLI's perspective, making future commands easy to add.

## `yarn workspaces list [flags]`

This command lists all of the packages for all of the workspaces alphabetically. If no workspaces are found it will error.

## `yarn workspaces run [flags] <command> ...`

This command runs a specific package script (as defined in the `scripts` property in `package.json`) for all of the packages for all of the workspaces.

Just like the `yarn run` command, any arguments after the command name will be passed as arguments to the package script.

For a _fail fast_ operation, we could traverse all the workspaces to check whether the specified command exists within that workspace, before we actually start executing. If not found, display an error with which workspace is missing that command.

The ordering of execution is also important. It must be executed _topologically_, so that it won't break the inter-dependant workspaces. From the Lerna documentation:

> By default, all tasks execute on packages in topologically sorted order as to respect the dependency relationships of the packages in question. Cycles are broken on a best-effort basis in a way not guaranteed to be consistent across Lerna invocations.
> 
> Topological sorting can cause concurrency bottlenecks if there are a small number of packages with many dependents or if some packages take a disproportionately long time to execute. The --no-sort option disables sorting, instead executing tasks in an arbitrary order with maximum concurrency.
> 
> This option can also help if you run multiple "watch" commands. Since lerna run will execute commands in topologically sorted order, it can end up waiting for a command before moving on. This will block execution when you run "watch" commands, since they typically never end. An example of a "watch" command is running babel with the --watch CLI flag.

The output should be prefixed with the name of the workspace (eg. `packages/package-a:`), so the user has a better idea of what is currently running.

### `--concurrency`

The `--concurrency` flag changes the number of child processes that are spawn when commands are run in parallel, defaulting to `4` (like Lerna).

### `--parallel`

If the `--parallel` flag is passed, it will runs the commands in parallel in separate child processes, instead of running them in series. This command will ignore the concurrency flag and topological sorting requirements. (Just like Lerna.)

## `yarn workspaces exec ...`

This commands runs an arbitrary shell command in each package. It is similar to `yarn workspaces run`, and respects the same flags, but instead of running a package script defined in `package.json` you can pass it arbitrary shell commands.

This is helpful for cases where you want to execute something that isn't worthy of storing in `package.json`, often when debugging or running a one-off.

For example:

```
$ yarn workspaces exec babel --out-dir ./lib ./src
```

## Common Flags

### `--packages`

The `--packages` flag takes a package name or glob, and it restricts the command being run to only take affect in those packages. 

**Note:** this flag operates on the package names, as defined by the `name` field in `package.json` files.

For example:

```
$ yarn workspaces list --packages 'babel-*' build
```


### `--workspaces`

The `--workspaces` flag takes a workspace name or glob, and it restricts the command being run to only take affect in those workspaces. 

**Note:** this flag operates on the workspace names, not the package names. For example `packages/my-package` would be a workspace name. This is helpful when working with multiple directories of workspaces.

For example, with a `workspaces` setup of:

```json
[
  "packages/*",
  "services/*",
  "utils/*"
]
```
```
packages/
  package-a/
  package-b/
  ...
services/
  api/
  cdn/
utils/
  ...
```

You could restrict the command to only run in `services/api` and `services/cdn by doing:

```
$ yarn workspaces run --workspaces 'services/*' start
```

# How We Teach This

Since this is a new feature and doesn't affect existing functionality, it won't change any existing user behavior. However, documentation should be added to explain the new features to those who want to opt-in to them.

# Drawbacks

This will increase the complexity of Yarn, instead of letting it live in Lerna and work in tandem. Especially while executing the package scripts, because we need to keep track of the package dependencies. 

# Alternatives

There's no current alternative using Yarn. You have to use Lerna and Yarn in concert, which causes confusion as to which one to use when.

# Unresolved questions

- Should package names and workspace names be handled? or only packages?
- What does Yarn define as a single "workspace"?
- Should parallel execution be handled?
- Should an `exec` command be added as well, similar to Lerna?
