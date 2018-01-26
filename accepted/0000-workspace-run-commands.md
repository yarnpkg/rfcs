* Start Date: 2017-09-21
* RFC PR:
* Yarn Issue:

# Summary

Allow Yarn CLI to execute a package script on each packages from the workspace root.

# Motivation

"Lerna". Lerna does a great job for handling monorepos. Since Yarn has a built-in
workspaces feature, we could use some of the functionalities that being used in
lerna (not entirely though!) for handling the monorepo more efficiently.

Just like installing all dependencies from one place (workspace root), it'd be great
to execute all the sub-package scripts from the workspace root using a single command.
Not only this would make a less back-and-forth traveling between the packages to
execute scripts, but also it'll help a lot for CI/CD configuration.

# Detailed design

As @BYK mentioned [here](https://github.com/yarnpkg/yarn/issues/4467#issuecomment-330873337),
we could create two commands for this feature.

* `yarn workspaces list`
* `yarn workspaces run <command>`

By introducing `workpaces` command, it brings a modular approach from the CLI's perspective.

## `yarn workspaces list --args`

This command will list out all the packages for all the workspaces (alphabetically). If no workspace is found, this will fail with error.

Additionally, we can pass a `--filter` argument followed by a workspace name. In that case, it'll list out all the package inside that specified workspace.

## `yarn workspaces run <command>`

This command will execute the specified command in all the packages under the workspace. Before we run the scripts, it should make sure that all packages has the script specified in the package.json. Otherwise, it should throw an error specifying which pacakge is missing that script.

Just like the `list` command, by default, it'll select the first workspace in case of no workspace name is provided.

The important thing here to note that the ordering of the execution. It must be executed in the same dependency order. This will assure that we could use a single "build" that builds inter-dependant packages without any problem.

# How We Teach This

Since this is a new feature and doesn't affect existing functionalities, it shouldn't affect existing/new users. However, a slight documentation should be written under the workspace CLI.

# Drawbacks

Complexity. Especially while executing the package scripts. We need to keep track of the package dependencies. And also, there's already a library (lerna) does the same thing and is compatible with yarn without any extra configuration.

# Alternatives

Don't have an alternative way (I can think of) inside yarn.

# Unresolved questions

How to handle multiple workspaces?
