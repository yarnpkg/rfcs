- Start Date: 2017-06-19
- RFC PR:
- Yarn Issue:

# Summary

Bring the scripts that run on certain commands closer to parity with NPM.

# Motivation

There are a few changes to the scripts hooks that NPM has made recently, and it
would be good to bring `yarn` to the same standard.

This means that `yarn` can remove hooks that don't make sense, and stop
supporting issues brought up by people confused about *why* a certain script is
running.

Additionally, by tracking closely to what NPM is doing with their hooks, `yarn`
remains a drop-in replacement, as most people expect it to act.

# Detailed design

### State of Scripts

Below are all the scripts that NPM currently auto-runs, and at which points they
run. Currently, there is only one deprecation that motivated this RFC, but it is
anticipated that there will be more in the long run.

This list is kind of long, because I've enumerated all options.

- `prepublish`
  - before `publish`
  - before `install`
    - *Note: this is the deprecation. `prepublish` will no longer run on
      install*
    - This hook should *not* be added, as it is removed as of `npm@5`.
- `prepare`
  - before `publish`
  - before `install`
  - *Note: This is the replacement for the current `prepublish` behaviour.*
- `prepublishOnly`
  - before `publish`
- `prepack`
  - before `pack`
  - before `publish`
- `postpack`
  - after `pack`
  - after `publish`
- `publish`
  - after `publish`
    - confusing, but runs *after* the package has been published.
- `postpublish`
  - after `publish`
- `preinstall`
  - before `install`
- `install`
  - after `install`
    - confusing, but runs *after* the package has been installed.
- `postinstall`
  - after `install`
- `preuninstall`
  - before `uninstall`
- `uninstall`
  - before `uninstall`
    - confusing, but runs *before* the package is uninstalled
- `postuninstall`
  - after `uninstall`
- `preversion`
  - before `version`
- `version`
  - after `version`
  - runs *before* the version commit is made
- `postversion`
  - after `version`
  - runs *after* the version commit is made
- `preshrinkwrap`
  - before `shrinkwrap`
- `shrinkwrap`
  - before `shrinkwrap`
    - confusing, but runs *before* the shrinkwrap is created
  - I'm not sure `yarn` needs to include this, given that `shrinkwrap` is not
    implemented.
- `postshrinkwrap`
  - after `shrinkwrap`

The following commands are backed by user-written scripts. The `pre` and `post`
commands are run before and after the user-written version, and there is no
built-in run.

*Note: `test` has a built-in default of `echo 'Error: no test specified'`, so
the `pre` and `post` will run regardless*

*Note: all other scripts will error, and no hook will run, except `restart`
(explained below).*

- `pretest`
  - before `test`
- `posttest`
  - after `test`
- `prestop`
  - before `stop`
- `poststop`
  - after `stop`
- `prestart`
  - before `start`
- `poststart`
  - after `start`
- `prerestart`
  - before `restart`
- `postrestart`
  - after `restart`

`restart` is a special case, because it will run the `stop` and then the `start`
scripts if `restart` doesn't exist. It doesn't throw an error if any of those
scripts are not defined. Interestingly, it will run the `pre`- and `post`-
scripts for `stop` and `start`, even if `stop` and `start` themselves are not
defined.

### Multiple Hooks, One Command

There are some commands that have *multiple* hooks attached to them. These hooks
will run in a certain order.

#### restart

`restart` is arguabled the most confusing behavior. First, let's look at the
behaviour when no `restart` script is defined.

Regardless of the definition of the actual `start` and `stop` commands,
`restart` will run the lifecycle hooks. If all lifecycle hooks are defined, the
scripts are run in the following order. If a particular script is not defined,
it is simply skipped.

`prerestart` -> `prestop` -> `poststop` -> `prestart` -> `poststart` ->
`postrestart`

This makes more sense when you look at the fact the `restart` runs `stop`, and
then `start` if there is no `restart`. It makes less sense that these hooks are
run if there is no `stop` or `start`, even though running `start` will error,
and run no hooks if `start` is not defined.

If `restart` is defined, the hooks are run like:

`prerestart` -> `restart` -> `postrestart`

#### publish

Due to the addition of the extra hooks to try and solve the confusion around the
`publish` hook, the hooks are run in the following order:

`prepublish` -> `prepare` -> `prepublishOnly`

In practice, the `prepublishOnly` event may be dropped at any time, so this hook
ordering makes little-to-no sense, given that `prepublishOnly` now has the same
behaviour as `publish`, yet they on different sides of `prepare`.

The current proposal would be to match the existing behaviour of `npm`. That
will also require changing the current `yarn` behaviour, as `prestart` and
`poststart` scripts currently run, even without a `start` defined.

# How We Teach This

This is a continuation of both `npm` patterns and existing `yarn` patterns. I
think that this could also be cleared up by displaying what hooks will be run at
the commencement, so that people can clearly see what is going to happen.

E.g.

```
> yarn start
Running prestart -> start -> poststart
```

And show only the hooks that get run, and what order they'll be run in.

# Drawbacks

- It will cause changes to the current `yarn` behaviour of running hooks even if
  no script is defined
  - e.g. `prestart` and `poststart` run, even without `start` being present.
    `npm` throws an error.
- It may change peoples existing workflows if they expect `prepublish` to run on
  `install`
  - However, this change brings `yarn` into line with `npm`

# Alternatives

- Split away from current `npm` behaviour
  - This gives `yarn` the option to define behaviours in a more modern way, and
    the flexibility to change when/how hooks run
- Leave the current behaviour as-is

# Unresolved questions

- How to display this change to users?
- Where to keep a list of hooks and what orders they run in?
