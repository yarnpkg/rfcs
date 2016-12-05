- Start Date: 2016-12-05
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

> Allow declaring dependencies in custom named groups (i.e. besides `dependencies`, `devDependencies`, ...)

Instead of restricting dependency declarations to the pre-existing groups it would be beneficial to allow developers to decide for themselves how they want to organize their declarations.

E.g. this:

```json
{
   "dependencies": {
     "bluebird": "3.3.5",
     "backbone": "1.3.3",
     "handlebars": "3.0.2",
     "lodash": "3.10.1"
   },
   "devDependencies": {
     "babel-core": "6.7.4",
     "eslint": "2.11.1",
     "handlebars-loader": "1.0.2",
     "less": "2.6.0",
     "less-loader": "2.2.2",
     "mocha": "2.4.5",
     "must": "0.13.1",
     "nightwatch": "0.8.18",
     "proxyquire": "1.7.4",
     "require-directory": "2.1.1",
     "sinon": "1.17.4",
     "webpack": "1.12.13",
     "webpack-dev-server": "1.14.1"
   } 
}
```

Could be re-organized into something like:

```json
{
   "dependencies": {
     "app": {
       "bluebird": "3.3.5",
       "backbone": "1.3.3",
       "handlebars": "3.0.2",
       "lodash": "3.10.1"
     },
     "build": {
       "babel-core": "6.7.4",
       "handlebars-loader": "1.0.2",
       "less": "2.6.0",
       "less-loader": "2.2.2",
       "webpack": "1.12.13"
     },
     "QA": {
       "eslint": "2.11.1"
     },
     "unittest": {
       "mocha": "2.4.5",
       "must": "0.13.1",
       "proxyquire": "1.7.4",
       "require-directory": "2.1.1",
       "sinon": "1.17.4"
     },
     "uitest": {
       "nightwatch": "0.8.18"
     },
     "dev": {
       "webpack-dev-server": "1.14.1"
     }
   } 
}
```

**Where the various group names (`app`, `build`, `QA`, ...) are developer-defined, i.e. they could be any string value.**

Installation of the packages could be as simple as passing multiple group names to the command-line:

```sh
$ yarn --dev --app --QA
```

Which means all dependencies declared in the `dev`, `app` and `QA` groups get installed. This allows us to combine groups as we see fit.

E.g on a testing server we could run

```sh
$ yarn --app --build --unittest
```

# Motivation

Currently dependencies can be declared into 4 groups/types:

* dependencies
* devDependencies
* peerDependencies
* optionalDependencies

While this made sense a few years ago, due to how development and deployment processes have evolved this feels insufficient in some workflows and methodologies.
Especially in a multi-tier architecture, where code gets installed in various stages and environments, the current way of organizing dependency declarations is messy, opaque and leads to wasteful installations.

The main benefits would be:

 * More efficient installs, i.e. install only what's necessary in the given context/environment, leading to:
    * (sometimes dramatic) speed improvements at install-time
    * less disk space waste.
 * Clearer application dependency structure.
 * Truly optional dependencies.
 
Though disk space comes cheap, this does have an impact when working with container technologies like docker. The above improved organization would allow creating specific environment targeting images and layers more easily. 

The currently incorrectly named `optionalDependencies` aren't optional at all. They get automatically installed unless the underlying OS doesn't support them. However, there are many use cases for truly optional dependencies: plugins, adapters, etc. 

There seems to be some support for similar concepts (though not this specific API proposal) in the community:

* https://github.com/npm/npm/issues/13479
* https://www.npmjs.com/package/npm-dep
* https://www.npmjs.com/package/dependency-groups
 

# Detailed design

My proposal is to:

1/ expand the `dependencies` entry in a `package.json` file to allow organizing the dependencies into user-defined (with "user" being the developer in this case) groups.

A valid `dependencies` entry would either have the current, existing format:

```json
{
  "dependencies":{
    "<dependency name>": "<version range || url>"
  }
}
```

**Or** the newly proposed:

```json
{
  "dependencies": {
    "<group name>": {
      "<dependency name>": "<version range || url>"
    }
  }
}
```

No nesting would be allowed (groups in groups in groups).

2/ Further more I'd like to propose a single pre-defined "magic" group name: `default`:

```json
{
  "dependencies": {
    "<group name>": {
      "<dependency name>": "<version range || url>"
    },
    "default": {
      "<dependency name>": "<version range || url>"
    }
  }
}
```

The `default` dependencies would always get installed, see below.

3/ Installation would allow passing these group names as switches to the command-line client, denoted with double dashes (`--`):

```sh
$ yarn --<group name> --<group name> ...
```

None, one or multiple group names could be passed, making all of these valid:

```sh
# install only the "default" dependencies
$ yarn

# install the "default" dependencies and those grouped as "a"
$ yarn --a

# install the "default" dependencies and those grouped as "a", "b" and "c"
$ yarn --a --b --c
```

In case a `default` entry is not declared and no group names are passed to the command-line client an appropriate warning should be thrown.

4/ Adding a dependency would use the same switches as above:

```
# add package "foo" to the "a" dependency group
$ yarn add foo --a
```

However in this case only one switch would be allowed.

```
# WARNING would be thrown
$ yarn add foo --a --b
```

5/ It would NOT be allowed to mix the proposed group dependency declarations with the old style pre-defined top-level group types:

```json
// INVALID
{
  "dependencies": {
    "<group name>": {
      "<dependency name>": "<version range || url>"
    }
  },
  "devDependencies": {
    "<dependency name>": "<version range || url>"
  }
}
```

I.e. either you stick with the npm-compatible dependency declarations -or- you use the new style.


# How We Teach This

Documentation and education wise this would be an addition, in the sense that all existing documentation would still be valid. 

Anybody familiar with "dependencies" and "devDependencies" would pick this up pretty fast, I think.

# Drawbacks

The major drawback would be that `package.json` files utilizing the new dependencies declaration style would no longer be recognized as valid manifest files by the `npm` CLI. I.e. once you go down this road, you can't use npm for dependency management in your project anymore.

# Alternatives

An alternative would be to simply allow extending the existing predefined top-level dependency groups with a similar format for custom groups. E.g.:

```json
{
  "dependencies": {
    "bluebird": "3.3.5",
    "backbone": "1.3.3",
    "handlebars": "3.0.2",
    "lodash": "3.10.1"
  },
  "buildDependencies": {
    "babel-core": "6.7.4",
    "handlebars-loader": "1.0.2",
    "less": "2.6.0",
    "less-loader": "2.2.2",
    "webpack": "1.12.13"
  },
  "QADependencies": {
   "eslint": "2.11.1"
  },
  "unittestDependencies": {
    "mocha": "2.4.5",
    "must": "0.13.1",
    "proxyquire": "1.7.4",
    "require-directory": "2.1.1",
    "sinon": "1.17.4"
  },
  "uitestDependencies": {
    "nightwatch": "0.8.18"
  },
  "devDependencies": {
    "webpack-dev-server": "1.14.1"
  }
}
```

The "default" group would not be necessary, since anything under "dependencies" could be regarded as such.

Installation and addition would still be the same:

```
# add package "foo" to the "a" dependency group
$ yarn add foo --a

# install all dependencies defined in "dependencies" and "a"
$ yarn --a
```

# Unresolved questions

* Should the format used in `package.json` be explicitly defined? 
* Maybe the community prefers the alternative style?