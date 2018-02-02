- Start Date: 2018-02-02
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

This adds a new command (name TBD, but e.g. `yarn assert-consistent`) which
indicated if `yarn` needs to be run to make `node_modules` consistent
with `yarn.lock`.

# Motivation

This is a common situation.

> User A: things aren't working correctly.
> User B: I updated the deps, did you run `yarn` after pulling?
> User A: Oh, nope.

This should be solved in a way that doesn't actually do anything
automatically, like running `yarn` via git hooks. Instead, it guides the
user to manage their own dependencies.

A common issue is that running `yarn` clears out symlinks, and if it's
fully automated, you might not remember to add the symlinks back. Another
is switching to a branch, only to make a documentation change, where
running `yarn` would be a waste of time.

Instead it's up to the automation to call this command at the appropriate
times, and either print a warning, or prevent an action from completing.
The user also needs to be able to override the check in cases where it'd
normally prevent an action from completing.

# Detailed design

In the simplest case, the command would do one of two things:

1. The deps are up to date. Print nothing, exit with status code 0.
2. The deps are outdated. Print an error, exit with a non-zero status code.

For example, this could be worked into a yarn script in package.json.

```json
  "scripts" {
    "build": "yarn assert-consistent && webpack"
  }
```

The command would accept a `--warn-only` flag that results in a status code
of 0 in all cases. This can also be set with an env var (name TBD), allowing the
user to override it without changing any files.

The simplest implementation would be hashing `yarn.lock` and storing the hash
in a file in `node_modules`. When `node_modules` is updated, we update the hash.
Not sure what edge cases this wouldn't cover.

In addition to the command line version, it should be published as an npm package,
allowing both a sync and async version for use in JavaScript-based scripts. The
sync version would return undefined or throw an error, the async would do the same
but wrapped in a `Promise`. The sync version could be a single line of code, with
both a `require` and call to the exported function.

# How We Teach This

I think the key is coming up with a good, self-evident name for the command,
and well written output in the fail case. The error output would include
the env var that skips the check. This gives users two clear solutions:

1. Run `yarn` to bring `node_modules` to current with the `yarn.lock`, the 95%
case.
2. Run the command again with an env var, skipping the check.

This is an opt-in feature, and one that needs to be used immediately for it
to impact a project.

While this feature impacts many other commands, it's nearly invisible to the
user of those commands. As such, it only needs to be mentioned on the new docs
page for this command. Users experiencing the error/warning this command produces
will be pointed to the relevant page.

In the design proposed here, there's no configuration option to enable it, which
would make it less clear on what's causing it. It's always an explicit command.

For existing users, I think we let it gain use organically. Projects by people
following yarn development will use it. Users will encounter the warning and read
about it in the docs. Then they'll start using it in their own projects, and the
chain continues from there.


# Drawbacks

A project that decides to use this feature may break other tooling.

Compatibility with `npm` is unclear (see unresolved questions).

The proposed mechanism for evaluating this isn't resistant to direct modifications
to `node_modules` (or modifications by npm), leading it to potentially return false
positives and false negatives. The issue is that this command should be *very* fast
to run. Any time we add to the commands this guards will have real impact on yarn users.

# Alternatives

There could be a config option that enables this check on all package.json scripts,
which would reduce a lot of boilerplate, but there would be no way to disable it
for specific scripts, and wouldn't apply to scripts not run by yarn, e.g. a `Makefile`,
bash script, or `.js` script.

# Unresolved questions

What should the command name, the flags, and the env var be named?

What should the error output look like? Should it include any non-static
information, e.g. a list of packages that would be changed by running
`yarn`?

What happens if the user runs `npm install` instead of `yarn`, and then
runs this command?

What happens if they run `yarn`, then `npm install foo`, then this command?

