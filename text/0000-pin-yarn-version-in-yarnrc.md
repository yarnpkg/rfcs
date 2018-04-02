- Start Date: 2017-09-12
- RFC PR: 
- Yarn Issue: 

# Summary

Since different versions of yarn have different guarantees as to how `node_modules` are installed, allow a project to select
what version of yarn should be used for that project.

# Motivation

Currently when running yarn, the system warns the user whenever the version
of yarn is not the latest version. However, there may be reasons _not_ to use
the latest version (e.g. incompatibility with your CI system), or errors that
are [introduced by a new version](https://github.com/yarnpkg/yarn/issues/4404)
of yarn that impact your project.

Instead, a project should be able to designate a version of yarn that all
developers should be using for that project and it should warn them if they
are using a version of yarn that is not the same as what that project
expects.

# Detailed design

Add an entry into `.yarnrc` which specifies the version(s) that are acceptable to be used, perhaps using the same format as `package.json` itself, e.g.:

```
project-version ^1.0.2
```

If the user is running `yarn install` without the `--prod` flag, and there is
a version-mismatch, yarn should generate a warning and tell the user to
download the latest acceptable version. If `yarn install` is running _with_
the `--prod` flag, and there is a version-mismatch, yarn should error and
quit (stating that there is a version-mismatch).

# How We Teach This

**A common criticism of yarn is that it [does not guarantee determinism between
different versions of yarn](https://news.ycombinator.com/item?id=14479693)
and there is no way to determine in a given development team if any individual engineer is using the correct version of yarn. Now if a larger team wants
to guarantee that, they can.**

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing npm patterns, existing Yarn
patterns, or as a wholly new one?

**The term `project-version` might make sense here.**

**What could certainly be difficult here is that users tend to install yarn
globally, so having different yarn-versions installed for different projects
could be challenging. This could be mitigated by intalling different versions
of `node` using `nvm` with different yarn versions.**

Would the acceptance of this proposal mean the Yarn documentation must be
re-organized or altered? Does it change how Yarn is taught to new users
at any level?

**We may need to change how we recommend installing yarn.**

How should this feature be introduced and taught to existing Yarn users?

**Would need to amend https://yarnpkg.com/lang/en/docs/configuration/**

# Drawbacks

_Why should we *not* do this? Please consider the impact on teaching people to
use Yarn, on the integration of this feature with other existing and planned
features, on the impact of churn on existing users._

**There could be serious issues with regards to having different projects that
people are working on that require different yarn versions. In this case, the
user will have to have a better way of selecting between current active yarn
version than our current options.**

_There are tradeoffs to choosing any path, please attempt to identify them here._

**The tradeoff is that we are highlighting a problem with yarn determinism and
compatibility, but that we are introducing a feature that directly addresses
 it.**

# Alternatives

_What other designs have been considered? What is the impact of not doing this?_

**We could ensure that new minor/patch versions of yarn never have any bugs and
that they always guarantee determinism of package installation between
versions. Alternatively, maybe we introduce "LTS" versions of yarn which only see bug-fix releases and do not change how packages are installed.**

# Unresolved questions

_Optional, but suggested for first drafts. What parts of the design are still
TBD?_

**Not clear how we would recommend switching between yarn versions.**