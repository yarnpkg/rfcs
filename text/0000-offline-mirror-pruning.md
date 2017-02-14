- Start Date: 2017-02-11
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

It would be helpful to have a built-in method to remove unneeded tarballs
from an offline mirror.

# Motivation

`yarn add` and `yarn remove` keep `package.json`, `node_modules`, and
`yarn.lock` in sync, so when a package is removed, it is removed in all three
places if possible. The same is not true for an offline mirror. When
a package is removed, it does not get deleted from the mirror, even if no other
package depends on it.

This behavior would be desirable in an environment where many projects share
the same offline mirror. However, when an offline mirror is only used by one
project, it would be reasonable to trim tarballs from the mirror when they are
no longer required. Developers would be able to keep the offline mirror as
small as possible, which is particularly beneficial when the mirror is checked
into source control.

# Detailed design

The feature can be controlled through a new configuration setting that turns on
automatic pruning. When `yarn-offline-mirror-pruning` is set to `true`, `yarn`
will check the offline mirror whenever `yarn.lock` is changed. If a package
exists in the mirror but no longer exists in `yarn.lock`, the package will be
deleted from the mirror.

Setting `yarn-offline-mirror-pruning` to `false` should result in the current
behavior (no pruning), which would also be the default behavior.

# How We Teach This

This feature should be presented as an extension to the current workflow for
maintaining an offline mirror. With a proper setup, the developer does not need
to worry about adding a package to the mirror. `yarn add` handles that work.
Similarly, with pruning turned on through the configuration, the developer
would not need to worry about removing packages from the mirror.

We would want to add an explanation for this feature to the existing
documentation. It would not be a priority to teach this feature to
brand new Yarn users since it is not critical functionality and only applies
to people who want to use an offline mirror.

# Drawbacks

Turning on the feature may add non-neglibile processing time for certain `yarn`
commands if the offline mirror is large.

In a monorepo setting, one project that turns on pruning may accidentally
wipe out a significant portion of a shared offline mirror.

# Alternatives

Users could just let their offline mirrors grow indefinitely.

They could also write their own script that parses `yarn.lock` and removes
uneeded packages from the offline mirror.

# Unresolved questions

Do we need to worry about a project turning on the feature when using a shared
offline mirror?

Should we also add a new flag to the CLI commands to do pruning on demand rather
than only being able to rely on automatic pruning? I can't think of a good use
case for only wanting to do pruning sometimes rather than always or never.
