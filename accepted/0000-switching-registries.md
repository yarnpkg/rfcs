- Start Date: 2017-09-21
- RFC PR:
- Yarn Issue:

# Summary

Yarn should always use the registry configured by the current user, falling back to the default registry otherwise. It should not use the registry specified in the `resolved` field of lockfile.

# Motivation

The default registry used by Yarn is `registry.yarnpkg.com`, but the use of alternate registries is possible through configuration or command-line flags. Using an alternate registry can be useful for a number of reasons; it can improve performance, facilitate serving private packages, and improve the reliability of build and test servers. However, many of these use cases are environment specific. These benefits cannot be realized without allowing the use of different registries in different environments.

* Using a registry mirror that is closer to you can dramatically improve performance. Using a single registry mirror might work well for teams that work in the same office, but teams that span great distances will need to use different registry mirrors to see the same benefit.
* Build environments can benefit from using a dedicated registry. In addition to the performance benefits, it can also protect against outages that might affect the public registries.

Currently, Yarn will only use the configured registry for newly installed packages. For packages that are listed in the lockfile already, Yarn will use the registry saved in the lockfile instead, ignoring the registry configuration of the current user. This effectively prevents switching registries between environments.

# Detailed design

Yarn should adopt the behavior `npm` introduced with `npm@5`, which is to always use the registry configured by the current user. This change was described in [this blog post](http://blog.npmjs.org/post/161081169345/v500):
>If you generated your package lock against registry A, and you switch to registry B, npm will now try to install the packages from registry B, instead of A. If you want to use different registries for different packages, use scope-specific registries (npm config set @myscope:registry=https://myownregist.ry/packages/). Different registries for different unscoped packages are not supported anymore.

Yarn already supports switching to a different registry and scoped registries. The change would be to use them in all cases rather than just for new packages.

### What about the `resolved` field in the lockfile?

The `resolved` field will not be used to determine which registry is used. Changes to to the `resolved` field, such as removing the registry, are outside the scope of this RFC.

### How do scoped registries work?

Yarn supports configuring a registry for a specific scope (e.g. `yarn config set '@foo:registry' 'https://registry.foo.com'`). If a scoped registry has been configured, then this registry shall be used for **all** packages under that scope. Using different registries within a single scope is not supported.

### Can non-scoped packages use alternate registries?

No, non-scoped packages would all use the same registry. This restriction simplifies the configuration, and keeps us in line with `npm`.

### Should the `resolved` field be used as a fallback?

No, the user's configured registry should be used at all times. Falling back to `resolved` could hide problems (i.e. outdated or misconfigured mirrors) and negatively affect performance. It could also leak information to third parties about which packages are being used, which might be undesirable in certain environments.

# How We Teach This

We should improve the documentation for configuring Yarn, with a focus on the registry and scoped registry configuration. This change should also be featured prominently and explained in the release notes, and any blog post or announcement accompanying the version this ships in. This is a significant change, but it does bring it more in line with the behavior of `npm`, which many people are familiar with.

Overall, the behavior of Yarn should be easier to understand after this change. The behavior will be more transparent and controllable by the user.

# Drawbacks

1. This would be a breaking change.
2. Alternate registries for non-scoped packages (or within scopes) would be impossible
While this was never supported officially, it was possible to do by directly editing the `yarn.lock` file. With this change, that would no longer be possible.

# Alternatives

1. Add additional configuration for overriding registries
We could preserve the existing behavior, but add additional configuration for overriding the registry. This `override-registry` would behave like `registry` configuration does in the main proposal. This would allow users to opt-in to the new behavior without making a breaking change.
While this would solve the problem, it would also make the configuration more confusing and difficult to teach.
2. Use the `registry` configuration only as a fallback to the `resolved` field
This could allow the install to succeed in cases where the `resolved` field has a registry that is inaccessible to the current user. However it would do nothing to address the use case where an alternate registry is used for performance reasons.

# Unresolved questions

* What is the performance impact of not using the cached tarball URL (`resolved`)?
