- Start Date: (2017-05-09)
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Lockfile yarn.lock should not include base registry(`https://registry.npmjs.org`).

# Motivation

In yarn.lock, the `resolved` field includes registry such as `https://registry.npmjs.org`.
In China, most of developers will set registry to `https://registry.npm.taobao.org` for speed, but for travis-ci, circleci, it seems to be slowly.

# Detailed design

Replace `resolved` by a `hash` field.
The `url` in `resolved` is unnecessary, and keeping `hash` is enough.

# How We Teach This

Just set registry if you do not want to use `https://registry.npmjs.org` before `yarn install`.
Or use `yarn install --registry=https://registry.npm.taobao.org`.

# Drawbacks

There is a lit of work for the users who really need the whole `resolved` field in their project.
**Example**
`yarn install --registry=https://registry.npm.taobao.org`

# Alternatives

Don't change the lockfile, but the real registry can be changed by `yarn install --registry=https://registry.npm.taobao.org`

# Unresolved questions

No questions
