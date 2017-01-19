- Start Date: 2016-10-12
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Original issue: https://github.com/yarnpkg/yarn/issues/904

From tweet: https://twitter.com/rstacruz/status/786052262841896960

Integrate a license checker as a yarn command.

> I have published a standalone package that does this: https://github.com/behance/license-to-fail

Yarn has `yarn licenses ls`. It would also be useful to know if certain packages
don't satify your license (or other similar files) requirements rather than just a list of them.

```bash
$ yarn licenses check
yarn licenses v0.14.0
Disallowed Licenses
├─ a-pkg@1.0.3
│  ├─ License: not-allowed-license
│  └─ URL: git+https://github.com/pkg/here.git
# error
```

# Motivation

Most apps/projects have certain assumptions about the kinds of dependencies they bring in.
Even if you check each new dependency, the dependencies of those dependencies may have issues.
There's isn't an easy/manual way to do this outside of checking the license of all dependencies.

It's most likely that there aren't issues but having an command to do so would allow running it on CI
just like a linter. Issues can be caught automatically and with confidence.

This solves the problem of checking the license of a new dependency brought in through a new PR
or of an existing package updating it's license (whether it's a direct dependency or indirect dependency).

The outcome is that users could run the command to notify which packages are disallowed.

# Detailed design

The basic idea is straightforward: Given an array of packages and their licenses, match that against an array of
licenses that are disallowed. If any match, error and print them out.

It would be useful to have a way to make a list of exceptions for when you want to whitelist a properitary pacakage.

In reality you will probably need to make a lot of exceptions for packages since not all projects have a license
or the program to check what license a project is doesn't always work.

## Exceptions/What to do with packages that have an "unknown" license

- the license checker isn't able to figure out the license
  - license is in the readme or some other form (not in package.json)
  - the license is correctly updated in master on git but not published (not maintained)
  - future version of the package has a license but it's an indirect dependency

The way license-to-fail does it is let you pass in a config file.

```bash
$ ./node_modules/.bin/license-to-fail ./path-to-config.js
```

The config file is just an object with a list of `allowedPackages` and a list of `allowedLicenses`.

```js
module.exports = {
  allowedPackages: [
    {
      "name": "allowed-package-name-here",
      "extraFieldsForDocumentation": "hello!", // optional
      "date": "date added", // optional
      "reason": "reason for allowing" // optional
    }
  ],
  allowedLicenses: [
    "MIT",
    "Apache",
    "ISC",
    "WTF"
  ],
  warnOnUnknown: true
};
```

# Alternatives

Just use a separate package rather than making it built-in like https://github.com/behance/license-to-fail already is (and others).

# Unresolved questions

How do users specify the allowed licenses and exceptions (differences for apps/libraries)?

- use package.json config
- infer from the package's own license which licenses would be acceptable
- use an yarnrc config
- use cli arguments for options

Should it warn or error with unknown licenses?
