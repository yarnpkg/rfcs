- Start Date: 2017-08-08
- RFC PR:
- Yarn Issue:

# Summary

Clarify what is a *hosted* git dependency, which dependency patterns should be considered as *hosted*, and what are the differences with a regular git dependency.

# Motivation

*Hosted* git dependencies are dependencies hosted on Github, Gitlab or Bitbucket that can download the manifest and dependency content with a HTTPS request instead of using the `git` command.

Because they are treated differently, they do not benefit from all the features of a regular git dependency (like yarnpkg/yarn#3553), or introduce some incompatibilities (yarnpkg/yarn#3923).
Also these differences are poorly documented and hard to identify from the code.

The **main goal** is to ensure that the same result can be expected by installing a *hosted* git dependency or a regular git dependency.

Secondly, exhaustively document which patterns are resolved as *hosted* dependencies, what are the implications for private repositories, and how to write a pattern to be explicitly *regular*.
Plus it would be nice to synchronise with `npm` (npm/hosted-git-info ?)

At last, as much as possible, let the feature be extensible to other git hosts in the future (for example, self-hosted gitlab or bitbucket).

# Detailed design

As a first step, the existing implementation for *hosted* git dependencies should be removed, and patterns that previously matched a hosted dependency should resolve to a git repository.

<small>Note: the alternative presented below would stop at this point.</small>

When matching a *hosted* dependency pattern, the implementation must be able to resolve the host, user(team) name and repository name. 
The implementation must then provide the following urls:

- url to download the git refs (branches and tags)
- url to download the archive at a specific commit
- url to download a file at a specific commit (used to download `package.json` or `yarn.json`)
- a fallback git repository url (to be used if previous urls are not accessible, for example permission issue)

The `Git` utility should be the only place to make a distinction between *hosted* repositories and other repositories.\
If it is a *hosted* repository, the `Git` utility should first try to make HTTPS requests, then fallback on using the `git` command.

**TBD** Behaviour of `git import` with hosted dependencies.

The following patterns should resolve to *hosted* dependencies:

- `user/repo` (Github shorthand)
- `protocol:user/repo` (protocol one of `github`, `gitlab`, `bitbucket`)
- `git@hostname/user/repo` (hostname one of `github.com`, `gitlab.com`, `bitbucket.org`, `bitbucket.com`)

The `.git` extension is optional.

**TBD** What about `https://github.com/user/repo`, `git@hostname:user/repo`

**TBD** Should `git:` protocol force a not-hosted mode? 

# How We Teach This

Because all git dependencies will result in the same installed content regardless or the pattern (*hosted* or not), and some special cases are being removed, it should not be necessary to introduce the change to new or existing Yarn users.

For advanced Yarn users, the documentation should describe the optimisations made for *hosted* dependencies, and the matching patterns.

# Drawbacks

The generated content in the lockfile and manifest will probably differ from the previous behaviour, that may break third-party tools.

While the new implementation should be compatible with the currently installed packages, some edge cases of the old implementation may not be identified and become incompatible.

# Alternatives

- *What other designs have been considered?*
  
  Resolve *hosted* git patterns to regular git patterns and consider all git dependencies as being *regular*.
  It lacks optimisations, but doesn't require extra documentation and, maybe, can handle private repositories in a simpler way. It shares the same other drawbacks.

- *What is the impact of not doing this?*

  Unexpected and/or undocumented behaviour

# Unresolved questions

- What will happen with private repositories (should fallback to `git` command)
- What should be stored in the manifest and lockfile
- Behaviour of `yarn import`
- Exhaustive list of *hosted* dependency patterns
- Differences with `npm`
- How to differentiate the hash from the commit number and the hash from the downloaded tarball or git archive. (Should probably only use the commit number)
