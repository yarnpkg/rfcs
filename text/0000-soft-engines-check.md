- Start Date: 2017-06-14
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Make the `engines` field in `package.json` produces soft warnings instead of hard
errors during `yarn install`/`yarn`.

# Motivation

See [#3430](https://github.com/yarnpkg/yarn/issues/3430#issuecomment-303392013).
`npm` has deprecated the `engineStrict` field in `package.json` since v3, and all
`engines` setting violations will only produce warnings
<sup>[[1]](https://github.com/npm/npm/releases/tag/v3.0.0)</sup>
<sup>[[2]](https://docs.npmjs.com/files/package.json#engines)</sup>; Yarn, on the
other hand, still throw hard errors:

```shell
error some-package@3.1.13: The engine "node" is incompatible with this module. Expected version "v7.6.0".
error Found incompatible module
```

This has caused considerable inconveniences due to several reasons:

* People download npm packages to include part or whole of them in client-side
codes; checking against the version of the downloader in this case doesn't make
any sense

* Oftentimes an `engines` setting violation doesn't prevent a package from working
properly

* Once there is a dependency that requires an `--ignore-engines` flag to install,
all subsequent installing actions will require the flag as well, making the hard
error losing its intentions and gravitas

This is also a frequently brought-up issue; the following is a list of
still-unresolved entries of this issue & dupes:

[#3282](https://github.com/yarnpkg/yarn/issues/3282)
[#2342](https://github.com/yarnpkg/yarn/issues/2342)
[#2172](https://github.com/yarnpkg/yarn/issues/2172)
[#1552](https://github.com/yarnpkg/yarn/issues/1552)
[#1363](https://github.com/yarnpkg/yarn/issues/1363)
[#1102](https://github.com/yarnpkg/yarn/issues/1102)

# Detailed design

### `yarn install`/`yarn`

| Before | After |
| :----: |:-----:|
| Hard errors on `engines` setting violations | Soft warnings on `engines` setting violations |

### `--ignore-engines` flag

| Before | After |
| :----: |:-----:|
| Ignore engines check | (Deprecate) |

### `yarn config set engine-strict true [--global]` config

| Before | After |
| :----: |:-----:|
| (N/A) | Hard errors on `engines` setting violations |

This is modeled after `npm config set engine-strict true`; if people would like
to enforce stricter settings it's very likely they'd want to have it as a config
instead of a flag.

# How We Teach This

Release note, deprecation warning, and blog post.

# Drawbacks

It may end up tripping someone if they didn't see the warning text during
installation process.

# Alternatives

Soft warnings for dependencies and hard errors for devDependencies. This however
doesn't align with `npm`, and the inconsistent behaviors may create further confusion.

# Unresolved questions

N/A
