- Start Date: 2018-02-12
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Focusing on a single workspace should be as easy as working on all workspaces together. There should be an easy way to install sibling workspaces as regular dependencies rather than being required to build all of them just to work on a single workspace.

# Motivation

Currently, yarn workspaces are optimized for people making changes across multiple projects in the 
monorepo, but they can make things more difficult if you want to focus on a single workspace. While
the automatic linking and auto-installation of dependencies for all workspaces can be convenient if you are actually working
on all workspaces, having to build and install everything locally for multiple projects when you only work on one
at a time can be time consuming compared to just pulling those packages down from a registry.

If you could focus on a single workspace as easily as if it were its own repo, the transition to a monorepo
from separate repos would be seamless for people who focus on a single workspace/repo at a time, and one of the
biggest reasons not to use workspaces would be removed. 

# Detailed design

Add a new flag for `yarn install`, `--focus`. The flag can only be used from an indvidual (target) workspace (not the root of a workspaces project or an isolated project). Instead of a flag, this could also be a new command, `yarn focus`. This has a minimal effect on the design.

With the focus flag, in addition to normal installation behavior, yarn install "shallowly" installs any sibling workspaces that the target workspace depends on. A shallow installation can be best explained with an example.

There is a monorepo with packages A and B, where A depends on B, and B depends on Foo, which is external and not part of the monorepo. If you try to focus on A, `yarn install --focus` will install everything that `yarn install` does now. In addition, it will shallowly install B in A/node_modules. Shallow installation means installing B but not installing any of its transitive dependencies (such as Foo). A regular install would already guarantee that Foo would be at root/node_modules, so regular node module resolution would already be able to find it and it is not needed under A.

The version of B that will be shallowly installed is the version that is in B's package.json (but downloaded from the registry). However, one edge case is when A depends on a different version of B than what is in B's package.json. This may happen in a scenario where A rolls back its version of B due to a bug in B but continues to release new versions of A. In this scenario, the shallowly installed version of B should be the version that A specifies, not the version of B's package.json. (This is the current behavior and should not change).

Another edge case is when A also depends on Foo, but depends on a different version than B. In this case, B's version of Foo should be installed under A/node_modules/B/node_modules/Foo. This will allow it to use its version of Foo without interfering with the version A uses (whether that is under root/node_modules/Foo or A/node_modules/Foo).

The most complicated edge case comes into play when there are local changes to B's package.json that have not been published to the registry. The problem with this is that package name and version are no longer a unique identifier for a copy of a package, which is currently relied on in many places. Using the local version of B is not an option because a difference in dependencies would break the remote version of B. Trying to install both the local and remote versions of B would require major changes to differentiate between 2 copies of the same package with the same version. Instead, we should use the remote version of B to determine all dependencies that are installed, rather than the local version.


The focus flag should prevent writing to the lockfile because the potentially different dependencies locally versus what is published should not change the lockfile. The integrity file should be marked with a flag to signals that a focused install was done to allow for quick bailouts on repeated focused installs and to prevent early bailouts if install is later run without the focus flag.

This approach should allow for quick switching between focused work and non-focused work, since every dependency stays in the same place and only a few extra ones are added or removed. (a regular yarn install should erase any shallow installations and let you go back to normal cross-package work).

# How We Teach This

*What names and terminology work best for these concepts and why?*
A new documentation section should be added for the focus flag. When explaining what it does, the term "shallow installation" should paint a clear picture of how sibling workspaces are installed.

*How is this idea best presented?*
As a way of enabling focused development in a single workspace without removing the ability to manage the whole
workspace from a subfolder. 

*How should this feature be introduced and taught to existing Yarn users?*
Add a new section for the focus flag in the install documentation. A blog post would be a good way to announce the feature as well.

*Would the acceptance of this proposal mean the Yarn documentation must be
re-organized or altered? Does it change how Yarn is taught to new users
at any level?*
Just adding new documetation for the new flag (see above). 

# Drawbacks

You have to rerun focus every time you run yarn add/upgrade. (This would also be the case for yarn install if focus was a command instead of a new flag). This can possibly be mitigated with future additions of focus flags for add and upgrade.

Focused installs still to have to install all dependencies for all repos (trying to ignore other repos would greatly increase complexity). If you are working on package A, which depends on B but does not depend on C, you still have to install C's dependencies (at the root, like normal) even though you don't need them. In practice, this should not be a huge problem because workspaces already optimizes away the need to reinstall common dependencies between C and A/B, which limits the unncessary installations. Additionally, if you have already run a regular install, the unnecessary dependencies will already 
be in place.

# Alternatives

As described above, focus could be a new command rather than a flag for the existing install command. However, its similarity to install makes it a better candidate for a flag rather than a whole new command. Flags are also easier to add auotmatically in .yarnrc files if you want to always run focused installs without having to think about it.

Instead of only shallowly installing sibling workspaces, focus could do a full install of a workspace's dependencies inside its node modules folder, allowing it to ignore the root node_modules. Similar flags could also be added to other commands, such as add and upgrade. However, in addition to being MUCH slower, it greatly increases the complexity of the code for install because the lockfile (which is shared across all workspaces) would need to be maintained correctly even though only a single workspace is being considered.

An alternative that would require no work would be to just encourage people to disable the workspaces feature
when they want to focus on a single workspace. This is a poor user experience though, and the feature flag for workspaces 
will likely be deleted at some point.
