- Start Date: 2019-01-07
- RFC PR:
- Yarn Issue:

# Summary

This RFC relaxed the requirement to install package into local `node_modules`,
to make it more friendly to layered file system that used by docker and
common CI cache system.

Note that although `npm` has `npm ci` that design for CI scenario, it does
not allow `npm ci` to run without `package.json`, this does not improve the
cachability of `node_modules`.

# Motivation

NPM projects model requires all project to specify their dependencies as well
as metadata, e.g. name, version, scripts, in one file called `package.json`.
However this is not cache-friendly with layered file system, which is mainly
used by docker. As content inside `node_modules` is depends on `package.json`,
any irrelevance change to `package.json`, e.g. a version bump, will invalidate
the cache of `node_modules`.

By allowing yarn to install packages only from a lock file, `node_modules` can
cached correctly. This helps docker to speed up building image.

# Detailed design

We could introduce a new command `yarn ci`, (or anything option else to `add`)
that install pacakges listed in `yarn.lock` found in current working directory.
Yarn will just downloading all packages, expanding and executing all hooks.
Consistency checks will be postponed till the time `pacakge.json` appears.

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
RUN yarn ci
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



# Alternatives


# Unresolved questions

