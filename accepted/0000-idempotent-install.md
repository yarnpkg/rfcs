- Start Date: 2016-12-13
- RFC PR: https://github.com/yarnpkg/rfcs/pull/37
- Yarn Issue: https://github.com/yarnpkg/yarn/issues/2241

# Summary

`yarn install` should be idempotent.

Ideally, the result of `yarn install` would not be statefully dependent on the contents of an existing `node_modules`, and would ensure the resulting `node_modules` is identical regardless of whether there is an existing `node_modules` or not. This seems very inline with the spirit of yarn's goal of deterministic builds (same `node_modules` independent of whether `node_modules` exists or the version of node that generated it).

# Motivation

Presently, a major headache with `npm` / binary `node_modules` (e.g., `heapdump`) is the need to manually run `npm rebuild` when upgrading node. Communicating this preemptively to developers prior to an upgrade is logistically very manual, leading to "Why is this broken for me?" when errors are not obvious (e.g., `Error: Cannot find module '../build/Debug/addon'`).

Since `yarn install` is near instant when dependencies are unchanged, having developers run `yarn install` after a `git pull` is no big deal. However, having developers regularly run `yarn install --force` with many dependencies is a non-starter (1s vs 100s).

# Detailed design

Assuming both a `package.json` and `yarn.lock` in the project's root...

*NOTE: A primed / clean yarn cache and/or `yarn-offline-mirror` are not applicable / relevant.*

**Path A (`node_modules` dne, node@X):**

- `yarn install` => binaries for node@X

**Path B: (`node_modules` installed w/ node@X, node@Y)**

- **Current, non-ideal**: `yarn install` => binaries for node@X
- **Ideal**: `yarn install` => binaries for **node@Y**


# How We Teach This

*What names and terminology work best for these concepts and why?*

 "`node_modules`" for node context, "rebuild" & "build" for npm legacy; "idempotent" for technical accuracy; "install" & "force" for yarn context (i.e., `yarn install`).

*How is this idea best presented?*

As a continuation of existing Yarn patterns: "deterministic builds".

*Would the acceptance of this proposal mean the Yarn documentation must be re-organized or altered?*

No.

*Does it change how Yarn is taught to new users at any level?*

Yes. This will all but eliminate the need to explain why rebuilds are needed after an upgrade.

*How should this feature be introduced and taught to existing Yarn users?*

By assuring users:

> `yarn install` ensures a consistent outcome.

No need to caveat ^ this with:

> Unless you upgraded node, then you need rebuild your binary modules with `yarn install --force`,
but don't worry about it reinstalling all your modules, even the non-binary ones.

# Drawbacks

Complexity of detection / knowing when to rebuild.

# Alternatives

Use a flag: `yarn install --check-rebuild` and/or support it in `.yarnrc` (`install-check-rebuild true`)
