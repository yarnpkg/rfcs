- Start Date: 2017-07-11
- RFC PR: 
- Yarn Issue: 

# Summary

To allow grouping of similar notifications with addition of categories.

# Motivation

As exhibited in [`yarn/issues/3738`](https://github.com/yarnpkg/yarn/issues/3738) not everyone perceives notifications at the same level of severity. Additionally, not all notifications are always clear at first glance. To help users quickly identify common issue types regardless of severity categories may be used. Categories provide an at-a-glace understanding of the reasoning for a notification and may be used for grouping like notifications.

# Detailed design

As [suggested in `yarn/issues/3869`](https://github.com/yarnpkg/yarn/issues/3869#issuecomment-314157889) an enumerable list of categories is provided for individual warning messages to make use of, enabling output similar to this:

<samp><pre>warn [infosec] vulnerability detected in ...
info [compat] kernel not supported in ...</pre></samp>

Where `[infosec]` and `[compat]` represent taxonomy terms specified in the list of available categories. If a category is not provided, a `[general]` category is output.

To retrieve an initial list of categories some croudsourcing may be done as part of this RFC. Categories should contain a `name` and a `label`, e.g.

```toml
[infosec]
label = "Information Security"
description = "Security-related notifications."
usage = "Use when a known security vulnerablility exists."

[interop]
label = "Interoperability"
description = "Notifications related to system compatablity"
usage = "Use when your kernel doesn't match your sanders."

[general]
label = "General"
description = "Ad hoc and unassigned notifications."
usage = "Use as a catchall or when categorization not provided."
```

This could later be extended to suggest severity levels for specific categories, helping normalize data.

# How We Teach This

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing npm patterns, existing Yarn
patterns, or as a wholly new one?

**We teach them by likening design with [Android push notifications](https://material.io/guidelines/patterns/notifications.html).**

Would the acceptance of this proposal mean the Yarn documentation must be
re-organized or altered? Does it change how Yarn is taught to new users
at any level?

**That's hard to say. Yes?**

How should this feature be introduced and taught to existing Yarn users?

**Some documentation might be nice. Beg brorrow and steal is my motto.**

# Drawbacks

_Why should we *not* do this? Please consider the impact on teaching people to
use Yarn, on the integration of this feature with other existing and planned
features, on the impact of churn on existing users._

Don't do this if the design becomes too complicated or it's not cross-packager compatible.**

_There are tradeoffs to choosing any path, please attempt to identify them here._

Trade-off is API surface increases, docs increase and disagreement may occur on proper cagegorization of notifications.

# Alternatives

_What other designs have been considered? What is the impact of not doing this?_

Other designs considered are more simple and do not allow for grouping. See [`yarn/issues/3869`](https://github.com/yarnpkg/yarn/issues/3869) for alternative solutions.

# Unresolved questions

_Optional, but suggested for first drafts. What parts of the design are still
TBD?_

Not clear how to expose API surface to package maintainers, though for Composer `extras` object in the package manifest may be a possiblity.