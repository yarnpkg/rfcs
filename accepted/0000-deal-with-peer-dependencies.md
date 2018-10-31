- Start Date: 2018-09-28
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Allow a project author to deal with peer dependency warnings, without the need
to install all dependencies, thus reducing warning fatigue and drawing attention
to new warnings.

_Throughout the document the term ‘developer’ will be used to indicate a project
author, i.e. the developer on a project that _consumes_ libraries vs a developer
of a library._

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
right now, they add these to the `ignore-warnings > missing-peer-dependency`
list of the _project’s_ `.yarnrc` configuration file.

For instance, as the project the developer is working on is a web project, they
want to always ignore a warning about the `react-native` peer dependency of
`@artsy/palette`:

```yaml
ignore-warnings:
  missing-peer-dependency:
    - name: "react-native"
      expected-by: "@artsy/palette"
      version: "*"
```

The `styled-components` dependency of the transitive `styled-reset` dependency,
however, is one that the developer cannot update to right now and so, _after_
having verified that the version that they can use works for their use-case, the
developer adds this warning to the ignore list:

```yaml
ignore-warnings:
  missing-peer-dependency:
    - name: "styled-components"
      expected-by: "styled-reset"
      version: ">=4.0.0"
```

----

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

----

Additionally, as is custom with build systems that allow control over how to
treat warnings, an option can be introduced to treat warnings as errors. This
means that developers on a project will be unable to accidentally introduce new
peer warnings, as installation will fail and they are forced to treat the new
failure by either satisfying the peer dependency or making the determination
that it should be ignore.

This should take the form of an additional `.yarnrc` entry to allow persistence
of this project into SCM and distribution across all developers on the project.

```yaml
treat-warnings-as-errors true
ignore-warnings:
  # ...
```

# How We Teach This

This change is only additive and does not change any existing behavior of Yarn
or how developers are supposed to interact with it. The new terminology clearly
communicates the intent and only introduces a negated version of existing
terminology.

Besides entries being added to the [the `.yarnrc` documentation][yarnrc-docs],
communication should also inlcude a blog post that explains the feature in a bit
more detail and stresses how this is a way for a developer to gain control over
their project’s dependency graph and ensure its stability.

Links to its documentation could be added to the existing documentation on [peer
dependencies][yarn-peer-deps], but otherwise does not need to be altered.

# Drawbacks

It’s one more configuration that needs to be documented and understood. On the
other hand, it is likely that the `ignore-warnings` project configuration will
be expanded upon in the future, such as ignoring a missing license key in a
dependency’s `package.json` file, and as such will be less of a niche
configuration.

# Alternatives

A previous itertation of this RFC suggested the addition of a key to the
`package.json` file instead, but moving this out to the `.yarnrc` file made it
clear that this is a project level concern, not a library one.

```json
{
    "ignoredPeerDependencies": {
        "@artsy/palette": ["react-native@*"]
    }
}
```

# Unresolved questions

The suggested format for the `ignore-warnings` list of the `.yarnrc` is YAML,
yet the existing configuration doesn’t appear to be strictly YAML.

----

As this new configuration deviating from the previous simple format, per the
above, does the ability to add entries to the `ignore-warnings` list need to be
exposed in the CLI, or even an interactive prompt, to ease working with it?

```
$ yarn config ignore-warning <type> [--name <name> --expected-by <expected-by> --version <version requirements>]
```

----

Should the `treat-warnings-as-errors` option also be exposed in the CLI? It
seems like when somebody wants to one-off fix warnings they can just look at the
output, whereas the point of this configuration is more to surface new warnings
in a continuous form.

[npm-peer-deps]: https://docs.npmjs.com/files/package.json#peerdependencies
[yarn-peer-deps]: https://yarnpkg.com/lang/en/docs/dependency-types/#toc-peerdependencies
[yarnrc-docs]: https://yarnpkg.com/lang/en/docs/yarnrc/
