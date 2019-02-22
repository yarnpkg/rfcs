- Start Date: 2017-05-09
- RFC PR:
- Yarn Issue:

# Summary

The lockfile yarn.lock should not include the base registry (`https://registry.yarnpkg.com`).

# Motivation

In yarn.lock, the `resolved` field includes registry such as `https://registry.yarnpkg.com`.

In China, most developers will set it to `https://registry.npm.taobao.org` for speed; but it seems slow for travis-ci and circleci.

By the way, the current approach leads to developers leaking their internal artifact repository sites to the public internet via yarn.lock if they have their company's artifact repository configured in a .npmrc or .yarnrc file.

# Detailed design

Replace the `resolved` by a `hash` field.
The `url` in `resolved` is unnecessary; keeping `hash` is enough. For example:

before
```
lodash@4.17.4:
  version "4.17.4"
  resolved "http://registry.npm.taobao.org/lodash/download/lodash-4.17.4.tgz#78203a4d1c328ae1d86dca6460e369b57f4055ae"
```
after
```
lodash@4.17.4:
  version "4.17.4"
  hash "78203a4d1c328ae1d86dca6460e369b57f4055ae"
```

# How We Teach This

Just set the registry before `yarn install` if you do not want to use `https://registry.yarnpkg.com`.

# Drawbacks
More effort is needed in order to support users who really need the whole `resolved` field in their project.

# Alternatives

Don't change the lockfile, but replace with the real registry by set registry config.

# Unresolved questions

How will this be rolled out to all Yarn using projects?

Will Yarn replace the entire yarn.lock file?

Will Yarn only use the new format for changed resolutions in the yarn.lock file?
