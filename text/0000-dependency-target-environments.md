- Start Date: 2017-01-30
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Support per-dependency target environments & record them in `package.json` file & `yarn` cli;

# Motivation

NPM package dependencies are typically categorized in 3 areas, which are defined in a `package.json`: 

```
  "dependencies": {
    "<dep>": "<version>"
  },
  "devDependencies": {
    "<dep>": "<version>"
  },
  "peerDependencies": {
    "<dep>": "<version>"
  }
```

Typically, when you install a dependency, you know where you intend to use it. Usually, this falls into a few categories, including, (but not limited to): `web` or `node` `native`. (see: https://webpack.js.org/configuration/target/#target)
 
This is satisfactory if you only have a single target environment. However, if you are installing dependencies for multiple targets, there are challenges:

* You are forced to mix dependences for these targets in a single `package.json`
* Analysis of dependencies becomes diluted and complex.
* Determining "where is this used?" is a guessing game.
* Limits potential automation routines that can benefit from more granular categorization of dependencies.

### Example scenario: Isomorphic web app
* Targets `web` & `node`.
* Dependencies:
	* `webpack`: targets `node`, for the building
	* `express`: targets `node`, for the server
	* `react`: targets `web` and `node`, for browser & server side rendering
* Challenges:
	* Using `DLLPlugin` & `DLLReferencePlugin` for webpack config. (see https://medium.com/@soederpop/webpack-plugins-been-we-been-keepin-on-the-dll-cdfdd6cb8cd7#.sgtvuxwl5)
	* This requires you to specify dependencies that are used in your target environment.
	* Automating this criteria is unrealistic, because your are limited to your `package.json`'s 3 dependecy types: `dependencies`, `devDependencies` & `peerDependencies`.
* Attempted solutions:
	* Get list of deps by reading `package.json`.
	* Get list of every folder `node_modules`. (yikes, now you're really in narnia)

# Detailed design

### Support `targets` property in `package.json` file
Allows for alias-based target definitions. Common target would probably be `node` & `web`. However, by allowing targets to be user-defined will allow for more flexibility.

#### Example
```
{
  "name": "my-app",
  "version": "1.0.0",
  "dependencies": {
    "express": "4.14.0",
    "react": "15.4.2"
  },
  "devDependencies": {
    "webpack": "2.2.0-rc.3"
  },
  "targets": {
    "node": ["react", "express"],
    "web": ["react"]
  }
}
```
 
#### Support `--target` option for following commands:
* `yarn add`
* `yarn remove`
* `yarn install`
* `yarn check`
* `yarn list`
* `yarn upgrade-interactive`

##### Usage
```
yarn add express --target node
yarn add react --target node,react
yarn add --dev webpack --target node

# expected package.json changes: 
# "express" added to dependencies, targets.node
# "react" added to dependencies, targets.node, targets.web
# "webpack" added to targets.node

```
```
yarn remove react --target node

# expected package.json changes: 
# "express" added to dependencies, targets.node
# "react" added to dependencies, targets.node, targets.web
# "webpack" added to targets.node

yarn remove react

# removes "react" from all dependencies & targets

```
```
yarn install --target web

# only installs dependencies defined in targets.node
```
```
yarn check --target web

# same current "check" behavior - but only for deps in targets.web
```
```
yarn list --target web

# same current "list" behavior - but only for deps in targets.web
```
```
yarn upgrade-interactive --target node

# same current "upgrade-interactive" behavior - but only for deps in target.node
```

# How We Teach This

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing npm patterns, existing Yarn
patterns, or as a wholly new one?

I believe this feature requires minimal explanation beyond the docs for `yarn cli`. Ideally, this feature would be opt-in only.

> Would the acceptance of this proposal mean the Yarn documentation must be
re-organized or altered? Does it change how Yarn is taught to new users
at any level?

I don't think so. There would be a need ot add to the docs for the feature. But, I don't think it would require altering current docs.

> How should this feature be introduced and taught to existing Yarn users?
I think the help provided via the `yarn cli` docs would be sufficient. 


# Drawbacks

Since I consider this feature to be opt-in only, I don't anticipate this being an intrusive feature. Thus, the only drawback I can think of would be the sheer fact of the complexity being added to yarn. I truly believe this is a feature that is lacking in web ecosystem currently.

# Alternatives

The other alternative I've convsidered is creating this feature and managing it per-project myself. However, I believe this responsibility lives closer to the realm of a package/dependency manager. I believe this is the next step for the future of dependency management in this ecosystem.

# Unresolved questions

* What other `yarn` cli commands could & should support this feature?
* What other challenges (a part from what I've descried above) could be solved with this feature?
