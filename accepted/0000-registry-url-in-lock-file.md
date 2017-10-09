- Start Date: 2017-09-21
- RFC PR:
- Yarn Issue:

# Summary

The yarn.lock file includes information about the registry that it was generated against. It would
be nice if there was no registry information associated with the lock file

# Motivation

We currently have two internal registries/mirrors. We generate the lock file against one we have in
our development environment. When our code is deployed to our build/testing environment, we
no longer have access to the original registry that was used to generate the lock file. This causes
our download to error as yarn cannot retrieve the packages.

The expected output is for yarn to recognise the registry URL supplied (via either cli flag or
`.yarnrc` or `.npmrc`) and download from that if the registry used in the lock file is not available.


# Detailed design

Let me know if this isn't enough, first time writing an RFC.

As a dev that generated their lock file against REGISTRY_A
I would like to pass a cli flag designating REGISTRY_B
And I would like yarn to download my dependencies from REGISTRY_B

As a dev that generated their lock file against REGISTRY_A
I would like to designate an alternate registry URL in my `.yarnrc` or `.npmrc` (REGISTRY_B)
And I would like yarn to download my dependencies from the alternate URL (REGISTRY_B)

I feel like the simplest solution would be to add a "hash" or "commit-ish" field to entries.

```
abbrev@1:
  version "1.1.0"
  hash "d0554c2256636e2f56e7c2e5ad183f859428d81f"
  resolved "https://registry.yarnpkg.com/abbrev/-/abbrev-1.1.0.tgz#d0554c2256636e2f56e7c2e5ad183f859428d81f"
```

This, with the version already provided, could then be used to generate a URL on the fly or at the beginning of operations if an alternate registry is specified in the scenarios above.

Or, the `resolved` field could change to not include the hostname/origin, which would then be inserted during operations. This would
probably have implications for backwards compatibility though.


# How We Teach This

This would largely be in the internal workings, and wouldn't require teaching. npm 5 does this
already, so it would be inherent.

> If you generated your package lock against registry A, and you switch to registry B, npm will now try to install the packages from registry B, instead of A. If you want to use different registries for different packages, use scope-specific registries (npm config set @myscope:registry=https://myownregist.ry/packages/). Different registries for different unscoped packages are not supported anymore.

http://blog.npmjs.org/post/161081169345/v500

# Drawbacks

I don't see any drawbacks in implementing this feature as it would keep track with npm

# Alternatives

Only workaround is to modify the yarn.lock file and replace all references to the registries with
the desired registry.

It seems that there are already many issues around this, so it seems there is a need.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
