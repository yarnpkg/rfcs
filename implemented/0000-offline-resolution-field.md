- Start Date: 28 Feb 2017
- RFC PR: https://github.com/yarnpkg/rfcs/pull/51
- Implementation: https://github.com/yarnpkg/yarn/pull/2970

# Summary

When enabling the offline mirror, Yarn updates the lockfile by stripping the registry URL from its `resolved` fields. This RFC aims to simplify this process by making such an update unneeded.

Related issues: https://github.com/yarnpkg/yarn/issues/393 / https://github.com/yarnpkg/yarn/issues/394

Tentative implementation: https://github.com/yarnpkg/yarn/pull/2970/

# Motivation

Yarn currently has two different type of values for the lockfile `resolved` fields:

  - When online, they're in the form `${source}/${name}-${version}.tar.gz#${hash}`

  - But when offline, they're instead `${name}-${version}.tar.gz#${hash}`

The current reasoning (or at least side effect) seems to be that it allows the fetch process to refuse installing things from the network when running under the `--offline` switch (and to always fetch things from the network otherwise instead of looking into the offline mirror). Unfortunately, such a separation also makes it harder to switch between working with a remote registry and an offline repository (for example, dev environments might not need the offline repository, but under the current design they can't do without).

Because of these reasons, it would be best for the `resolved` field to contain the same informations during both online and offline work, *as long as the files we fetch are the expected ones* (ie. their hashes match).

# Detailed design

I suggest the following:

  - Adding the `${source}` part to the `resolved` field even when offline

# Drawbacks

Nothing should break. More iterations will be required to address the other issues raised in the previous iterations of this document.

# How We Teach This

This change is quite transparent, since it's unlikely the users will ever want to update the yarn.lock file manually.

# Alternatives

  - Instead of adding the package registry to each `resolved` field, we could remove it instead.
