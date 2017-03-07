- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

When enabling the offline mirror, Yarn updates the lockfile by stripping the registry URL from its `resolved` fields. This RFC aims to simplify this process by making such an update unneeded.

Related issues: https://github.com/yarnpkg/yarn/issues/393 / https://github.com/yarnpkg/yarn/issues/394

# Motivation

Yarn currently has two different type of values for the lockfile `resolved` fields:

  - When online, they're in the form `${source}/${name}-${version}.tar.gz#${hash}`
  
  - But when offline, they're instead `${name}-${version}.tar.gz#${hash}`

The current reasoning (or at least side effect) seems to be that it allows the fetch process to refuse installing things from the network when running under the `--offline` switch (and to always fetch things from the network otherwise instead of looking into the offline mirror). Unfortunately, such a separation also makes it harder to switch between working with a remote registry and an offline repository (for example, dev environments might not need the offline repository, but under the current design they can't do without).

Because of these reasons, it would be best for the `resolved` field to contain the same informations during both online and offline work, *as long as the files we fetch are the expected ones* (ie. their hashes match).

# Detailed design

I suggest the following:

  - Removing the `${source}` part of the `resolved` field

    - **Rational: Why removing the source part of the resolved field?**
  
      The source is redundant, and misleading:
      
        - If we change our registry URL, the package itself will not have changed. The lockfile should not be modified.
        
        - The source is already in the package.json file - we can use it when we need to, since we will have checked by then that the resolved version still match (when working with a registry). Worst case scenario (direct link) we will catch that the file will have changed when checking the hash - but that's not something we could have avoided with storing the source in the lockfile anyway.

  - (Optional) Splitting the resulting `resolved` field into two others: `hash` (the hash of the tarballs) and `tarball` (the tarball name).

  - Adding the hash to the tarball name. This is to prevent issues where two projects are being worked on at the same time, both with the same dependency `foobar`, except that one of those projects is using the development branch of `foobar` (possibly fetching the project directly from git). In such a case, both projects lockfile would resolve to the same dependency name, the same dependency version (because maintainers don't usually bump version number until the release), but different hashes.

  - When installing a registry package that is specified in the lockfile but not present in the offline cache:

    - (Offline) We just throw.

    - (Online) We first check to see if the resolved version matches the package.json semver range. If it does, we download the package tarball that belongs to this specific version, validate it against the resolved hash, store it inside the offline cache, and use it to setup the dependency. If it doesn't, we can proceed as we currently do, updating the lockfile in the process to point to the new correct version.

  - When installing a non-registry (ie. git, direct url, etc) package that is specified in the lockfile but not present in the offline cache:

    - (Offline) We just throw

    - (Online) We should fetch the direct tarball (possibly by cloning and tarballing the referenced repository when working with git), and check that the hash matches the one in the lockfile. If it does, we can just store the tarball inside the offline cache, and use it to setup the dependency. If it doesn't, we can proceed as we currently do, updating the lockfile in the process to point to the new correct version.

# Drawbacks

It won't be possible to run `yarn install` and expect everything to work if the following conditions are met:

  - The package is only stored on a private registry, and not on the regular one

  - The user hasn't set the right private registry in Yarn

In such a case, we won't have access to the resolved registry location anymore, so we won't be able to query it directly. We will try to load the package from the regular registry, which will fail because it won't exist there.

# How We Teach This

This change is quite transparent, since it's unlikely the users will ever want to update the yarn.lock file manually. I feel like this behaviour is relatively intuitive: we first look into the lockfile, then into the package.json, and construct the cache without further interaction with the user.

# Alternatives

  - Instead of removing the package registry, we could add it on every resolution, even when offline. See above for why I think it would be a bad idea.

# Unresolved questions

An important question is the backward compatibility. The easiest way would probably be to strip the "resolved" fields while we read the yarn.lock file if we notice that they contain a registry URL. It would only require a minimal amount of legacy support codepath.
