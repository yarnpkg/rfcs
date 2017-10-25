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

The cli flag would be `override-registry`, to which an object can be passed to transform the registry requests.

The entry in the `.yarnrc` or `.npmrc` would also be `override-registry`

See "Detailed Design" for values it would receive.

### Example yarn entries

**Development Environment yarn.lock**
```
ansi-regex@^2.0.0:
  version "2.1.1"
  resolved "https://mydev.registry.com/ansi-regex/-/ansi-regex-2.1.1.tgz#c3b33ab5ee360d86e0e628f0468ae7ef27d654df"

ansi-styles@^2.2.1:
  version "2.2.1"
  resolved "https://mydev.registry.com/ansi-styles/-/ansi-styles-2.2.1.tgz#b432dd3358b634cf75e1e4664368240533c1ddbe"
```

**Build Environment yarn.lock**
```
ansi-regex@^2.0.0:
  version "2.1.1"
  resolved "https://mybuild.registry.com/ansi-regex/-/ansi-regex-2.1.1.tgz#c3b33ab5ee360d86e0e628f0468ae7ef27d654df"

ansi-styles@^2.2.1:
  version "2.2.1"
  resolved "https://mybuild.registry.com/ansi-styles/-/ansi-styles-2.2.1.tgz#b432dd3358b634cf75e1e4664368240533c1ddbe"
```

# Detailed design

As a dev that generated their lock file against REGISTRY_A
I would like to pass a cli flag designating REGISTRY_B
And I would like yarn to download my dependencies from REGISTRY_B

As a dev that generated their lock file against REGISTRY_A
I would like to designate an alternate registry URL in my `.yarnrc` or `.npmrc` (REGISTRY_B)
And I would like yarn to download my dependencies from the alternate URL (REGISTRY_B)

Based on feedback received, we shouldn't be modifying the yarn.lock file. Developers can provide
an `override-registry` option in either the cli or the `.yarnrc` or `.npmrc`

Example object would be `{"<generated-registry-entry>": "<registry-to-request-from>"}`

**Examples based on above [yarn entries](#example-yarn-entries)**

```
// cli
yarn --override-registry={"mydev.registry.com": "mybuild.registry.com"}
```

```
// .npmrc or .yarnrc
override-registry={"mydev.registry.com": "mybuild.registry.com"}
```

Using the example above, when this override is provided, the packages would get downloaded from "mybuild.registry.com"

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
