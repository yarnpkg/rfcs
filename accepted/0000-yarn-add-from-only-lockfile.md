- Start Date: 2018-12-12
- RFC PR:
- Yarn Issue:

# Summary

This RFC relaxed the requirement to install package into local `node_modules`,
to make it more friendly to layered file system that used by docker and
common CI cache system.

# Motivation

NPM projects model requires all project to specify their dependencies as well
as project metadata, e.g. name, version, scripts, in one file called
`package.json`. However this is not cache-friendly with layered file system
that used by docker. As `node_modules` is depends on `package.json`, any
irrelevance change to `package.json`, e.g. a version bump, invalidate the
cache of `node_modules`.

To speed thing up, by allowing install packages only from lock file,
`node_modules` can be cached even `package.json` is changed. Even if the lock
file is inconsistent `package.json`, we can use `yarn check` to find such
error.

# Detailed design

We could introduce a new option `--only-lockfile`(or anything else) to
`yarn add` that install pacakges listed in `yarn.lock` found in current
working directory. Yarn just bypass the consistency check between
`package.json` and `yarn.lock`, as user is expected to run `yarn check
--verify-tree` later.

A common Dockerfile of a node project is:

```Dockerfile
FROM node
ADD package.json yarn.lock ./
RUN yarn add --frozen-lock-file
ADD . .
```

Suppose this RFC have been implemented, Dockerfile can be changed to:

```Dockerfile
FROM node
ADD yarn.lock ./
RUN yarn add --from-lockfile
ADD package.json ./
RUN yarn check --verify-tree
ADD . .
```

The new approach is more complex, but cache is not invalidated as long as
no change to `dependencies` or `devDependencies` have been made.


# How We Teach This

No new concept or terminology will be introduced. Users who looking into this
feature shall already have some background of layered file system.

# Drawbacks

To be discuss

# Alternatives


# Unresolved questions

