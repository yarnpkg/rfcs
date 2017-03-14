- Start Date: 2017-03-14
- RFC PR:
- Yarn Issue: [yarnpkg/yarn#986](https://github.com/yarnpkg/yarn/issues/986)

# Summary

Allow users to bypass the global cache.

# Motivation

For certain situations, it would be preferrable to not use a global cache for
packages because the cache takes space and the step of copying from the cache
to `node_modules` can take a significant amount of time. Some examples are:

1. Building a Docker image. Images are isolated, so there's no point in having
a cache within the image. If anything, it has the negative effect of increasing
the image size.

2. Running Yarn in a CI context. This is another example of a potentially
isolated environment where we would just want to put the installed packages
directly into `node_modules`. While sophisticated CI setups might have
multiple projects share the same cache, there are certainly setups that don't
do that.

3. Running Yarn on a long-running server. The cache grows indefinitely over
time, eventually reaching max disk size or inode limit. Simply bypassing the
cache at run time could be preferrable to having to periodically run `yarn cache
clean`.

Overall, the desired effects of bypassing the cache are to increase the speed
and decrease the disk usage of certain Yarn commands (i.e., install, add,
upgrade).

# Detailed design

There should be a CLI switch `--no-cache` that tells Yarn to skip doing any work
with the cache.

Furthermore, there could be an equivalent `.yarnrc` setting of `no-cache` that
does the same thing when set to `true`. This way, the developer doesn't need to
remember to use the CLI switch every time.

# How We Teach This

We would update the [documentation](https://yarnpkg.com/en/docs/cli/cache) for
the `yarn cache` command.

# Drawbacks

Increases the complexity of the implementation of Yarn.

# Alternatives

In the linked issue, there is a suggestion to set the cache folder to some
in-memory or temporary location, but this solution is hacky and doesn't really
help with the speed concern because Yarn still has to do the work of copying
the packages to `node_modules`.

The same reasoning applies to the option of immediately running
`yarn cache clean`.

# Unresolved questions
