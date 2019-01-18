- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Inspired by [`npx`](https://medium.com/@maybekatz/introducing-npx-an-npm-package-runner-55f7d4bd282b), enhance `yarn <argv>` command to lookup global modules registry and prompt to install if there's no such module or it's outdated.

# Motivation

There are packages (CLIs) that both serve as global and local binaries, examples are `react-native`, `jest`, `eslint`, `ember-cli` to name a few.
With `npx` we can do:

```sh
npx react-native init MyApp
# fetching react-native, populating npm local cache
# running locally cached react-native init MyApp
```

But Yarn is not tied to Node foundation and it's not possible to introduce new binaries just like that but yarn-compatible.

However, Yarn has this great feature of using `yarn <argv>` command to run npm scripts or look up local node_modules to execute binaries. Seems like a natural evolution to expand this command to be even more powerful and search for binaries even further.

# Detailed design

We could use existing `yarn create` command that does similar thing (populate global node_modules and execute the package with `create-` prefix, just without this prefix)

When running `yarn package` the lookup would now look like this:

- check for `package` in `package.json` scripts, execute (current)
- check for `package` in local bin, execute (current)
- check for `package` in global bin (new)
  - __if__ latest version, print "running global module", execute, __else__
  - prompt to download new version,
    - __if__ confirm, execute, __else__
    - exit

## Example:

When there's no `react-native` global binary:

```sh
yarn react-native init MyApp
info: binary not found; use "react-native@0.59" from registry? (Y/n)
# fetching react-native, populating yarn global cache
# running global react-native init MyApp
yarn jest
yarn react-native run-ios # from local install
```

When latest version of binary is available:

```sh
yarn react-native init MyApp
info: using global binary of "react-native@0.59"
# running global react-native init MyApp
```

When binary is outdated

```sh
yarn react-native init MyApp
info: outdated "react-native@0.59" found; use "react-native@0.60" from registry? (Y/n)
# running global react-native init MyApp
```

# How We Teach This

Extend the docs on default `yarn` command.

# Drawbacks

- increased complexity of `yarn` command
- `yarn create` becomes obsolete (maybe an advantage though?)
- not yet sure why, but I'd assume people may like to opt out from this behavior, could be put behind a config flag


# Alternatives

Put this behavior behind another command, like `yarn gx package-name`
