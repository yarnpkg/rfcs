- Start Date: 2018-09-28
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Allow a developer to deal with peer dependency warnings, without the need to
install all dependencies, thus reducing warning fatigue and drawing attention to
new warnings.

# Motivation

[Peer dependencies][peer-deps] are a useful way of informing the developer that
a dependency relies on the host project to provide a transitive dependency
required by the dependency, without imposing that transitive dependency onto the
project’s dependency graph on its own.

However, there are legitimate cases when the developer may want to ignore a peer
dependency warning. Such as the host not being compatible with the dependency,
the version constraint being too restrictive, or otherwise not being interested
in the dependency.

While treating warnings as errors is often useful, warnings are specifically not
errors, because they could possibly be ignored after judgment by the developer.

Currently, the inability to deal with peer dependency warnings, other than
installing the required package/version, leads to warning fatigue, will be
ignored, and will introduce bugs.

_Regarding a version constraint being too restrictive, this usually comes up
when the transitive dependency’s version is lower than v1.0.0 and the dependency
has a version requirement on it using a semantic-version operator such as `^`.
E.g. `graphql@0.13.0` will not satisfy a requirement like `graphql@^0.12.0`._

# Detailed design

Consider the following peer dependency warnings shown on `yarn install`:

```
warning " > @artsy/palette@2.18.0" has unmet peer dependency "react-native@^0.55.4".
warning "@artsy/palette > styled-reset@1.4.0" has incorrect peer dependency "styled-components@>=4.0.0".
```

When the developer has determined that they do not want or cannot install these
right now, they add these to the `ignoredPeerDependencies` list of their
`package.json` manifest.

For instance, as the project the developer is working on is a web project, they
want to always ignore a warning about the `react-native` peer dependency of
`@artsy/palette`:

```json
{
    "ignoredPeerDependencies": {
        "@artsy/palette": ["react-native@*"]
    }
}
```

The `styled-components` dependency of the transitive `styled-reset` dependency,
however, is one that the developer cannot update to right now and so, _after_
having verified that the version that they can use works for their use-case, the
developer adds this warning to the ignore list:

```json
{
    "ignoredPeerDependencies": {
        "styled-reset": ["styled-components@>=4.0.0"]
    }
}
```

Afterwards, running `yarn install` shows no more warnings, until either:

* any dependency introduces a _new_ peer dependency

  ```
  warning " > react-tracking@5.3.0" has unmet peer dependency "babel-runtime@^6.20.0".
  ```

* _another_ dependency relies on `react-native`

  ```
  warning " > @artsy/emission@3.42.0" has unmet peer dependency "react-native@^0.55.4".
  ```

* the `styled-reset` dependency changes its requirement on `styled-components`

  ```
  warning "@artsy/palette > styled-reset@1.5.0" has an updated ignored peer dependency "styled-components@>=3.9.0".
  ```

In each of these cases, the developer’s attention should be drawn to the warning
so they can deal with it at the time they have the right context.

An additional warning could be shown when a listed (transitive) dependency no
longer exists in the dependency graph.

```
warning "styled-reset" with ignored peer dependencies is no longer depended on.
```

Finally, at any given time all warnings should be visible by passing the
`--verbose` flag to `yarn install`.

# How We Teach This

This change is only additive and does not change any existing behavior of Yarn
or how developers are supposed to interact with it. The new terminology clearly
communicates the intent and only introduces a negated version of existing
terminology.

Its communication should follow that of Yarn’s
[`resolutions` feature][yarn-resolutions], in that it is an extra way that a
developer can gain control over their dependency graph and ensure its stability.

Links to its documentation could be added to the existing documentation on [peer
dependencies][yarn-peer-deps], but otherwise does not need to be altered.

# Drawbacks

It’s one more configuration that needs to be documented and understood.

An alternative that does not introduce a new `package.json` key could be
preferable, but that does not appear to be feasible while maintaining
compatibility with npm.

# Alternatives

Introducing a new version requirement negation operator, for example:

```json
{
    "peerDependencies": {
        "styled-reset": "styled-components@!4.0.0"
    }
}
```

However, this breaks compatibility with npm and, more importantly, doesn’t allow
for multiple peer dependencies of the (transitive) dependency to be ignored.

# Unresolved questions

I don’t feel strongly about the configuration, except that I dislike more
configuration. Any suggestions that could lead to the same outcome with less
new configuration would be greatly preferred.

Perhaps the alternative design listed above could piggy-back on the
[`resolutions` key][yarn-resolutions]?

[npm-peer-deps]: https://docs.npmjs.com/files/package.json#peerdependencies
[yarn-peer-deps]: https://yarnpkg.com/lang/en/docs/dependency-types/#toc-peerdependencies
[yarn-resolutions]: https://yarnpkg.com/lang/en/docs/selective-version-resolutions/