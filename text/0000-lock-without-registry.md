- Start Date: 2017-05-09
- RFC PR:
- Yarn Issue:

# Summary

The lockfile yarn.lock should not include the base registry (`https://registry.npmjs.org`).

# Motivation

In yarn.lock, the `resolved` field includes registry such as `https://registry.npmjs.org`.
In China, most developers will set it to `https://registry.npm.taobao.org` for speed; but it seems slow for travis-ci and circleci.

# Detailed design

Replace the `resolved` by a `hash` field.
The `url` in `resolved` is unnecessary; keeping `hash` is enough.

# How We Teach This

Just set the registry before `yarn install` if you do not want to use `https://registry.npmjs.org`.
Or use `yarn install --registry=https://registry.npm.taobao.org`.

# Drawbacks
More effort is needed in order to support users who really need the whole `resolved` field in their project.

# Alternatives

Don't change the lockfile, but change the real registry by

`yarn install --registry=https://registry.npm.taobao.org`

# Unresolved questions

No questions
