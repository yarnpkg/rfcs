- Start Date: 2018-02-12
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Working on a workspace in isolation should be as easy as working on all workspaces together. Running a yarn command in
a workspace should behave the same whether it is part of a larger project or whether it is its own separate repo. There
should be a option on relevant commands, `--isolated` to have a command affect a single workspace instead of all workspaces.

# Motivation

Currently, yarn workspaces are optimized for people making changes across multiple projects in the 
monorepo, but they can make things more difficult if you want to work on a single workspace in isolation. While
the automatic linking and auto-installation of dependencies for all workspaces can be convenient if you are actually working
on all workspaces, having to build and install everything locally for multiple projects when you only work on one
at a time can be time consuming compared to just pulling those packages down from a registry.

If you could work on a single workspace in isolation as easily as if it were its own repo, the transition to a monorepo
from separate repos would be seamless for people who focus on a single workspace/repo at a time, and one of the
biggest reasons not to use workspaces would be removed. 

# Detailed design

Terminology used for this doc:
- rootDir: the root directory of a workspaces project. the one with the package.json with workspaces in it
- workspaceDir: Directory for an individual workspace/package
- isolated flag: `yarn foo --isolated`

In order to be able to interact with a single workspace in both isolation and in the context of the project as a whole,
relevant yarn commands should be able to be run with an `--isolated` flag.

Relevant commands (based on current functionality) include
- install
- add
- upgrade
- upgrade-interactive
- outdated
- check
- list
- test (more relevant to [run commands RFC](https://github.com/yarnpkg/rfcs/blob/master/accepted/0000-workspace-run-commands.md). see below)
- run (more relevant to [run commands RFC](https://github.com/yarnpkg/rfcs/blob/master/accepted/0000-workspace-run-commands.md). see below)

This list may grow in the future (for example if yarn adds the ability to publish all workspaces at once).

If you run a command with the isolated flag and are in a workspaceDir, it will run that command as if the workspaceDir you were currently in existed in isolation (with few exceptions detailed below).

If you run a command with the isolated flag but you are at the rootDir (or any other directory not underneath a workspaceDir), it will give an error saying that the isolated flag can only be used inside an individual workspace.

## Command behavior with isolated flag
### install
Install only that package's dependencies, and do not hoist them to the rootDir. Install dependencies on other
workspaces from registry instead of using the local version. 

It should still use the rootDir's lockfile since adding lockfiles for each workspace will cause conflicts
across workspaces. Since other workspaces will not be in the lockfile, it should determine their versions as if it was 
installing without a lockfile (by taking the newest version that matches their semver).

Since install will not take into account other workspaces, it would change the lockfile if allowed to. To avoid this, it should prevent lockfile changes as if `--pure-lockfile` was passed. This behavior should be documented clearly with the option.

### add
Add the dependency to the workspace's package.json and add update yarn.lock (at the root) to take this new dependency into account. Install the dependency (and any new transitive dependencies) in the workspace's node_modules folder. 

### upgrade
Upgrade all dependencies of the workspace in the root yarn.lock, respecting semvers of the current workspace and also
every other workspaces. Install all dependencies for the current workspace in its node_modules, not hoisting anything
to the root. Take the latest matching versions from the registry for all dependencies on other workspaces. Do not install dependencies for other workspaces that are not needed for the current one. 

### upgrade-interactive
Interactive version of an isolated upgrade. Bring up the interactive prompt, only showing the workspace's dependencies.
Record upgrades in the root yarn.lock. Install all dependencies for the current workspace in its node_modules, not hoisting anything
to the root. Take the latest matching versions from the registry for all dependencies on other workspaces.

### outdated
#### With isolated flag
Should show the list of packages that upgrade-interactive would show with isolated flag.

#### Changes to current behavior. 
None.

### check
Verify the workspace's package.json against the root yarn.lock file. Use the workspace node_modules (not hoisted) when looking
at actual package contents.

### list
List only dependencies needed by the current workspace.

# How We Teach This

*What names and terminology work best for these concepts and why?*
No new terminology is needed. The `--isolated` flag is just a new flag that certain commands recognize. The documentation should be able to give a good explanation of the behavior.

*How is this idea best presented?*
As a way of enabling isolated development in a single workspace without removing the ability to manage the whole
workspace from a subfolder. 

*How should this feature be introduced and taught to existing Yarn users?*
Add a new section for the isolated flag for the relevant CLI commands in the documentation, explaining how it behaves differently when run with the flag.

A blog post would be a good way to announce the feature as well.

*Would the acceptance of this proposal mean the Yarn documentation must be
re-organized or altered? Does it change how Yarn is taught to new users
at any level?*
Just adding new documetation for the new flag (see above). 

# Drawbacks

For projects where working in isolation is the common case, it's still a bit awkward to have to specify the isolated flag every time you run a relevant command. This can be mitigated by using yarnrc to default the flag to true for each workspace.

The logic for affected commands could get complicated as they would need to support 3 different ways of being run
- current behavior in workspaces
- isolation in a single workspace (where yarn.lock is at the root)
- current isolated behavior in non-workspace project (where yarn.lock is at the same level as package.json)

# Alternatives

Rather than installing dependencies in the local node_modules of a specific workspace when running a command with the isolated flag, they could be hoisted (where possible) like the default commands and then only locally install
dependencies on other workspaces (since the hoisted links can't be used). This was not chosen because while it would 
generally be faster (since most packages are already hoisted), it would be a worse simulation of isolated development
since the dependencies would not be in the workspaces's node_modules folder. Adding symlinks to node_modules for
all hoisted packages would be a middle-ground approach that came close simulating isolated development while still
keeping dependencies in a consistent place where possible.

An alternative that would require no work would be to just encourage people to disable the workspaces feature
when they want to work in isolation. This is a poor user experience though, and the feature flag for workspaces 
will likely be deleted at some point.

# Unresolved questions

Should we eventually make the commands that can be run in isolation context-aware and have them default to isolation if run in a single workspace? This would be a breaking change that would need to wait until yarn 2.0 and get feedback from the community before deciding. It can be decided separately from the rest of the proposal after it has been used in the real world.

Should using the isolated flag at the root directory just run with the default non-isolated behavior instead of erroring out (like described above).
