- Start Date: 2016-10-12
- RFC PR:
- Yarn Issue:

# Summary

Add symlink `link:` dependency type to enable complex cross-project development
workflows.

# Motivation

This RFC is a spinoff of yarn's [issue #884](https://github.com/yarnpkg/yarn/issues/884).

We've been using some kind of monorepo approach for our projects for quite some
time, a bit like [lerna](https://github.com/lerna/lerna) but with a more private
and nested approach (our packages aren't expected to be published for now) along
some specific needs: we must to link some public dependencies (eg. mongoose)
that must be the exact same instance accross our subpackages (otherwise you
encounter a lot of exotic bugs, edge cases), we also link our devDeps (they are
shared accross all our subpackages).

At first we used [linklocal](https://github.com/timoxley/linklocal) with npm@2,
leveraging a custom use of the `file:` prefix (basically just symlinking them),
but npm@3 broke a lot of things related to their handling (eg.
[#10343](https://github.com/npm/npm/issues/10343)). We ended up moving to
[ied](https://github.com/alexanderGugel/ied) where we implemented the `file:`
prefix handling using simple symlinks, that tackled our need.

I would love to be able to switch theses project to yarn but I would need a way
to create these links.

npm is also considering to add the same `link:` specifier, see this [recent RFC](https://github.com/npm/npm/pull/15900).

# Detailed design

We Add a new `link:` specifier that would just create symlinks and that's it
  (regardless of destination's existence)

I've already implemented the changes in yarn's [pr#1109](https://github.com/yarnpkg/yarn/pull/1109) and I'm
currently maintaining a fork since my team already rely on this for several
projects.

# How We Teach This

I think `link:` is pretty explicit, an update to the docs/cli-help should be
enough.

# Drawbacks

Not sure how exactly cross-platform symlinks are today. However it looks like
[Microsoft might be catching up](https://blogs.windows.com/buildingapps/2016/12/02/symlinks-windows-10/) on this issue.

# Alternatives

Beside the two alternatives exposed above, I can't think of anything else for
now.

Drawback of not implementing this is that we restrict how creative developers
can be with complex multi-packages workflows.

# Unresolved questions

Not sure how we should handle actually publishing packages with such
dependencies, the existing behavior for `file:` types?
