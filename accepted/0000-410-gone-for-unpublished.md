- Start Date: 2018-12-06 (fill me in with today's date, YYYY-MM-DD)
- RFC PR:
- Yarn Issue:

# Summary

Unpublished packages currently cause pain for yarn users. A recommended approach
ends up being to [delete yarn.lock and re-generate it](https://github.com/yarnpkg/yarn/issues/6702#issuecomment-442314783),
which can cause a huge amount of dependencies to change when only one was needed.

Unpublished packages currently return the HTTP status code "404 Not Found".
This code is designed for resources that might exist again in the future. In
the NPM ecosystem, version numbers are wisely
[immutable](https://www.npmjs.com/policies/unpublish). A better HTTP status
code to return for an unpublished package is "410 Gone", designed for resources
that will never come back.

When `yarn` encounters a [410
Gone](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/410) resource,
it could try to resolve the related semantic version again, possibly selecting a newer
version, fixing the issue the single dependency without requiring regenerating all of "yarn.lock".

# Motivation

Like many others I just lived through the pain of the
[Great Unpublishing of har-validator 5.1.2](https://github.com/yarnpkg/yarn/issues/6694).
I tried many things to get `yarn` to notice that this module was gone, and to re-resolve
The semantic version of `^5.1.0` to a newer version that existed, like `5.1.3`.  Finding
no workable solution, I searched online and found multiple people recommending to delete
`yarn.lock` and re-generate it.

I did that, and *many* unrelated dependencies got upgraded, causing new test failures. Pain.

# Detailed design

## Server Side

On the server side, this could be implemented in the Yarn NPM proxy, and possibly in the upstream
NPM registry as well.

The servers must already be tracking "unpublished versions" to prevent version number re-use.
So it seems like a small change on the servers to return 410 in this case as well.

## Experience during a yarn install

As a client, `yarn` can get into a state where the `yarn.lock` references a
file which is in the state `410 Gone` on the server.

During interactive installation, the user could be prompted:

> "The har-validator 5.1.2 dependency from your yarn.lock has been unpublished. Would you like to use a newer version instead?"

The user might be prompted to "Select minimum upgrade", "Select latest version" or "see all newer versions
and select".

It may also be better to passively notify the user and continue:

"warning: har-validator 5.1.2 has been unpublished. `har-validator ^5.1.0 is now being resolved to
`har-validator 5.1.3`.

The safest passive choice is likely the "minimum upgrade" to a version that exists.

We should also consider the case of the `event-stream` unpublishing, where the initial advice was
to *downgrade*. No newer version was available.

After the replacement version is selected, `yarn` would install the package and update `yarn.lock`
to reflect the change following the existing add/update code paths.

## Caching

The `410 Gone` module should be purged from the Yarn cache.

## We can't fix everything

If the semver refers to a specific version that version has been unpublished,
there no reasonable version to automatically select. We could prompt the user if they want
install the latest version (if any) with a warning that's not compatible with the `semver`.

# How We Teach This

The new behavior should be somewhat intuitive and user-friendly.

# Drawbacks

There may be some other clients that expect unpublished modules to return 404.
These clients could be confused by a "410 Gone" response. Failure to handle a valid
HTTP status code could be considered a bug in the other clients.

# Alternatives

`yarn` could be updated to handle 404s in a similar way, but this doesn't seem as safe.
A 404 may indicate a temporary issue, where changing the version of a dependency is not desirable.

# Unresolved questions

Why isn't "410 Gone" already more popular?
