- Start Date: 2019-09-03
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

This RFC attempts reduce disk usage, increase install speed, and improve multi-package project development. It proposes symlinking packages to a shared location, and in that location, using a relative symlink for node_modules as a method of indirection.

# Motivation

## Many packages

Many small packages in the Node.js ecosystem produces space-intensive installs and a large number of these installs. Relatedly, these installs take a significant amount of of time, even with a warm cache (depends on filesystem I/O).

## Multi-package projects

A monorepo with several concurrently versioned packages presents challenges.

Ideally, each package could declare its dependencies separately, receive only these dependencies and no more, *and* be linked to the other packages in the same repo (a change in package X is immediately realized by IDE and build tools for downstream package Y). It would "just work".

In practice, significant compromises have to be made, even with specialized monorepo managers like [Lerna](https://github.com/lerna/lerna).

# Detailed design

## Description

When `--link-shared` is `deps` or `all`, Yarn installs dependencies to a shared location. (`all` creates a node_modules symlink for the current package as well, in case it itself is depended. See Example 2.)

Each shared package uses a relative symlink for node_modules as a method of path-dependent indrection, allowing the use site to specify the actual resolved dependencies.

This install layout is only compatible with node when [`--preserve-symlink`](https://nodejs.org/api/cli.html#cli_preserve_symlinks) is used  (added in v6.3). 

## Example 1

Suppose the current package depends on pokedex@1.0.0 and svg@2.0.0. Suppose pokedex@1.0.0 depends on math@1.0.0 and svg@1.0.0.

With `--link-shared=deps` on the current package,

```
~/.yarn/
  v5/
    npm-math-1.0.0-abc1111111111111/
      index.js
      package.json
    npm-pokedex-1.0.0-abc2222222222222/
      index.js
      lib/
        util.js
      node_modules -> ../pokedex-abc2222222222222.node_modules
      package.json
    npm-svg-1.0.0-abc3333333333333/
      index.js
      package.json
    npm-svg-2.0.0-abc4444444444444/
      index.js
      package.json
```

```
index.js
node_modules/
  pokedex -> ~/.yarn/v5/npm-pokedex-1.0.0-abc2222222222222
  pokedex-abc2222222222222.node_modules/
    svg -> ~/.yarn/v5/npm-svg-1.0.0-abc3333333333333
  math -> ~/.yarn/v5/npm-math-1.0.0-abc1111111111111
  svg -> ~/.yarn/v5/npm-svg-2.0.0-abc4444444444444
package.json
server.js
```

## Example 2

In a monorepo, suppose package a depends on b, nasal-demon@1.0.0, trex@~1.1.0, and zombo@1.0.0. Suppose package b depends on trex@^1.0.0 and zombo@2.0.0.

With `--link-shared=all` on package a and b,

```
~/.yarn/
  v5/
    npm-nasal-demon-1.0.0-abc1111111111111/
      index.js
      package.json
    npm-trex-1.1.0-abc2222222222222/
      index.js
      package.json
    npm-trex-1.2.0-abc3333333333333/
      index.js
      package.json
    npm-zombo-1.0.0-abc3333333333333/
      index.js
      package.json
    npm-zombo-2.0.0-abc4444444444444/
      index.js
      package.json
```

```
a/
  index.js
  node_modules -> ../a.node_modules # perhaps needs a path hash too
a.node_modules/
  b -> ../../b
  b.node_modules/
    zombo -> ~/.yarn/v5/npm-zombo-2.0.0-abc4444444444444
  nasal-demon -> ~/.yarn/v5/npm-nasal-demon-1.0.0-abc1111111111111
  trex -> ~/.yarn/v5/npm-trex-1.1.0-abc2222222222222
  zombo -> ~/.yarn/v5/npm-zombo-1.0.0-abc3333333333333
b/
  index.js
  node_modules -> ../b.node_modules
b.node_modules/
  zombo -> ~/.yarn/v5/npm-zombo-2.0.0-abc4444444444444
```

## Comments

This install process is very small and very fast, it only has to create N directories and N symlinks, where N is the number of packages that are installed.

The symlink includes the package hash to account for aliases (e.g. `"pokedex2": "npm:pokedex2@2.0.0"`).

All hoisting, multiple dependency, dependency alias, and other paradigms continue to work.

The monorepo case is handled wonderfully.

As described in the examples, the symlinks in the shared install directory are dangling when accessed via the canoncial path, but if desired this is trivially remedied my adding empty directories.

# How We Teach This

## Shared install

A shared install is an installation performed according to the described paradigm.

## Shared installation directory

The shared installation directory is the common location for installed packages. Could be called "global" but this may be confusing with the existing unrelated `--global` concept.

The shared installation directory could be in the existing ~/.cache/yarn location, though "clear the cache" is not a harmless operation.

## Compatibility

This features does not conflict with existing yarn concepts, and can be opted into via RC file or command line flag.

The most important thing is adding `--preserve-symlinks` to node.

# Drawbacks

## pnpm

[pnpm decided not to symlink to the global store](https://pnpm.js.org/en/faq#why-have-hard-links-at-all-why-not-symlink-directly-to-the-global-store), and instead uses hardlinks.

The reasons given in the [Github discussion](https://github.com/nodejs/node-eps/issues/46) are outdated, and AFAIK this proposed design was not considered.

## Unintentional global changes

Any changes to the shared packages would be reflect across all installations. Local edits for debugging purposes would have unobvious effects. This could also be helpful.

## Clean cache

There is no way to clean the cache, without requiring other projects. 

## Changes runtime compatibility

This requires `--preserve-symlinks` to be specified at runtime.

## Compatibility

Some programs might react poorly to symlinks. Though unless they are processing the symlink specially, they should see the same layout as before.

## Complexity

This adds to the already numerous options yarn offers for installing packages. However, it should interact with these options in a straightforward manner.

# Alternatives

pnpm uses hardlinks. However, this is more file operations, and more signficantly it doesn't work well when adding files in the monorepo case.

# Unresolved questions

What happens on Windows?
