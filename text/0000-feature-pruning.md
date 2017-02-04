- Start Date: 2016-12-07
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

> Allow organizing dependencies in custom named groups

At different points in a package's lifecycle, the package may require a
different set of dependencies, currently represented as `devDependencies`,
`peerDependencies`, `dependencies` and `optionalDependencies`.  While these sets
of dependencies are expressive for the majority of existing node packages
published on the public registry, some packages, and many applications, have
lifecycles that are much more complicated than than the develop and deploy
lifecycle that currently exists.  This rfc allows for packages to define custom
dependency closures to match their lifecycle.

# Motivation

Better support for continous integration pipelines. Currently, there is a class
of bugs that is possible to write that cannot be caught with npm test. Consider
the following: you have a node module for which you've dutifully written unit
tests and listed your test framework as a dev dependency so that contributors
continuous integration server can build and test your package with the same
version of your build and test tools. A contributor accidentally lists a regular
(i.e. production) dependency as a dev dependency, and then opens a pull request.
Their tests work locally (since they ran an npm install and installed all the
dev dependencies) and your CI build also succeeds (to install your test
framework you have to install your dev dependencies). You merge the pull request
and publish a new version. The first time that the error can be detected (except
for human review of PRs) is when one of your clients first installs your new
version and gets a module not found error.

# Detailed design

There are two main parts to my proposed design: a new `feature` hash in
package.json, and a new command, `yarn prune --<feature-name>`.  When the
`--<feature-name>` or `--feature <feature-name>` flag is provided, the
dependencies listed in `packageJson.feature.<feature-name>` are retained, and
all other dependencies are removed.

## Package.json

I propose adding a new hash to `package.json`, called `features`.  An example
usage might be

```
{
  "dependencies": {
    "chalk": "*"
  },
  "devDependencies": {
    "ava": "*",
    "webpack": "*",
    "selenium-webdriver": "*"
  },
  "features": {
    "build": ["webpack"],
    "unit-test": ["ava"],
    "integ-test": ["selenium-webdriver"],
    "test": ["unit-test", "integ-test"]
  }
}
```

The features hash will contain an arbitrary number of user-defined keys which
correspond to flags passed to `yarn prune`.  The values associated with those
keys are always lists, consisting of either objects or strings.  If passed a
string, it must be one of the following: a key in the `dependencies`,
`devDependencies`, `peerDependencies` or `optionalDependencies`; the name of
another key in the `features` hash; or one of the strings `dependencies`,
`devDependencies`, `peerDependencies` or `optionalDependencies`.  If the
name of a dependency is provided, that dependency is a member of the feature; if
the name of a feature is provided, all dependencies in that feature are included
in the parent feature; if one of `dependencies`, `devDependencies`,
`peerDependencies` or `optionalDependencies`, all dependencies listed in the
corresponding hash of package.json are considered members of the feature.  This,
however, can lead to ambiguity; for instance if you want to have a feature be
the name of one of your dependencies (for instance if you have a dependency that
has peerDependencies and has a dedicated CI stage, like a a11y test).  In this
case, an object may be passed for disambiguation.  The object must contain two
keys, a `type`, which is one of { `feature`, `dependency`, `package.json` }, and
a `name`, which is the name of the feature or dependency, or the key to a
package.json field.  Thus, the following package.json is equivalent to the
example above

```
{
  "dependencies": {
    "chalk": "*"
  },
  "devDependencies": {
    "ava": "*",
    "webpack": "*",
    "selenium-webdriver": "*"
  },
  "features": {
    "build": [{"type": "dependency", "name": "webpack"}],
    "unit-test": [{"type": "dependency", "name": "ava"}],
    "integ-test": [{"type": "dependency", "name": "selenium-webdriver"}],
    "test": [{ "type": "feature", "name": "unit-test"},
             { "type": "feature", "name": "integ-test"}]
  }
}
```

## Yarn prune

I also propose adding a new cli command, `yarn prune`, named for the equivalent
[npm command](https://docs.npmjs.com/cli/prune), which it approximates.  For npm
compatibility, a `yarn prune`, with no arguments, would remove any dependencies
that are not needed.  In this way, it is similar to a `yarn install --force`,
except that it is strictly subtractive, rather than additive.  

However, unlike the existing npm command, when flags are passed to `yarn prune`,
the set of dependencies specified by the feature whose name corresponds to the
flag will be retained, as well as all transitive dependencies, while the rest of
the dependencies are removed.  In the example above, for instance, running
`yarn prune --build` would leave only the dependency "webpack" and its transitive
dependencies in `node_modules`.  Flags would be composable, and would retain the
_union_ of all features that appear in any feature passed as a flag.  In the
example above, running `yarn prune --build --unit-test` would leave "ava",
"webpack", and all their transitive dependencies on disk.  

To maintain compatibility with npm, and to make this easier to teach and use,
some flags would be reserved to match the existing dependency hashes: `--prod`
and `--production` for dependencies listed in `dependencies`; `--dev` for
`devDependencies`; `--peer` for `peerDependencies`; and `--opt` and `--optional`
for `optionalDependencies`.  If users would like to use features with one of the
reserved names, a `--feature` flag is provided.  The commands
`yarn prune --build --unit-test` and
`yarn prune --feature build --feature unit-test` would therefore be equivalent.

# How We Teach This

A useful start is to draw an analogy between the existing behavior of
`npm prune --production`, and the new features.  The analogy is imperfect, since
the existing production dependencies are stored in separate hashes, but is close
enough that it should be easy to understand.

Another useful resource is to draw an analogy with
[Rust's features](http://doc.crates.io/manifest.html#the-features-section),
which are a similar idea.

I don't think this would require a documentation re-organization; I think an
adding a doc on `prune` would be enough; if we feel strongly enough that this is
something to be recommended, an aside in the existing doc on `install` should
suffice.

# Drawbacks

The big drawback, to my mind, is that it is not future-proof in the least.  
Consider if npm decided to add a new dependency type, `testDependencies`, and
added an `npm prune --test`, or even just added an `npm prune --no-optional`.  
Any existing package with a "test" or "no-optional" feature would either obscure
the new npm behavior, or the yarn feature.  This downside could be significantly
lessened by only supporting the feature flag.

Another drawback is the api will become larger -- one benefit of moving to yarn
is that the api is smaller, and we'd be adding an entire new command, which
makes yarn seem bigger and scarier to new users.

Another drawback is that, no matter how we implement it, there will be some
ambiguity.  In particular, what happens when you perform two `prune`s in a row,
say an `npm prune --test`, followed by an `npm prune --dev`?  One might expect
that it would have the same effect as providing those two flags together in a
single command `npm prune --test --dev`, i.e. it would produce the union of
test and dev dependencies.  Another reasonable expectation (and what would
happen as proposed) is that you'd get the effect of the removal of all non-test
dependencies followed the removal of all non-dev dependencies, or the
intersection of the test and dev dependencies.  A non-trivial downside of this
rfc is that whichever behavior we chose to implement, someone will think its
wrong and we should have done the other one.

# Alternatives

We considered adding
[features to install](https://github.com/yarnpkg/rfcs/pull/35), but that would
require users to opt into an entirely new ecosystem, and would no longer be able
to operate in the npm ecosystem.

If we don't implement this, everything will probably be okay. Much like today,
every once in a while you'll install a package from the registry and get a
missing module error.  To work around this, you'll add a temporary dependency,
or downgrade to a previous version, or work off your own fork.  While you're
waiting for your CI to complete, you'll stare wistfully out the window and
wonder if supporting features would have prevented this, then forget about it
completely when your build passes, and you'll go home and have a nice tea and
be happy and fulfilled.

# Unresolved questions

Should we allow arbitrary flags at all, or should we only support the `feature`
flag?  If so, should we privilege the `--prod`, `--production` and `--dev`
flags, or is that needlessly complicated?  Should we also provide `--optional`
and `--opt`, or `--peer` for optional and peer dependencies?

Should we support a `feature` flag provided multiple times
(`yarn prune --feature list --feature of --feaure features`)? A `features` flag
provided once (`yarn prune --features=list,of,features`)?  Both?

Is there a better way to disambiguate composable features from packages than
types?

Should we only allow the `dependencies`, `devDependencies`, `peerDependencies`
and `optionalDependencies` hashes to be included as feature members, or
arbitrary package.json keys (for interoperability with other ecosystems that
store config in package.json, say `xo` or `jest`)?  If so, should we include
a plugin system for those ecosystems to translate their existing config to
dependency sets?

Should we add a feature column to `yarn outdated`, to indicate which features
the out-of-date dependency is a part of?

Should `yarn ls` support listing dependencies that are a part of a feature's
dependency closure?

Should `yarn why` include features?

Should `yarn prune` without arguments or flags continue to recommend
`yarn install --force` instead?

Should regular dependencies always be retained, even if the `--prod` flag is not
provided?

Should we add an `--exclude` switch for install, which would install all 
dependencies except for the ones declared in the named feature?
