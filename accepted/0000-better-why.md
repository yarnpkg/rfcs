- Start Date: (fill me in with today's date, 2018-08-01)
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

The command `yarn why` gives me the direct dependencies of a given package.
It would be much more useful to show the dependency chains leading to this package.

# Motivation

Knowing which direct dependencies cause a given package to be installed is
useful in several cases.

## Really understand why a dependency got installed

The current version of `yarn why` does not really tell why a dependency is installed,
you will need to run it recursively.
The current version of the command is scoped to the package name. When two versions
of the same package are installed, I would like to understand why a given version is installed.

## Update vulnerable package

A vulnerability in a package I use has been discovered, I need to upgrade this
dependency to a later version. However this dependency is not a direct dependency listed
in my package.json files. I would like to know ALL the direct dependencies that I
need to update/remove to get rid of this vulnerable package.

## Dedup packages

A given package is installed with two different versions. I would like to find out
why and what packages I need to upgrade to remove this duplication. I would like
to do this so I don't ship to customers two versions of the same code.

# Detailed design

The implementation would be quite simple. We traverse up the dependency chains
leading to the required package. While doing so, we print the packages to screen.
If the package is a direct dependency (declared in one of our package.json) then mention
it. This should work with yarn workspaces (it is even more useful there).
Note that a dependency can be both direct and indirect at the same time, the tool should make this visible.
For each package, both the requested version (eg. ^1.2.3) and the resolved version (eg. 1.2.5)
should be displayed. The same resolved version can match several requested versions 
(eg. ^1.2.3 and ^1.2.4 can both resolve to 1.2.5), this fact should be clear when displaying
the dependency chain.

## Example

### Repo structure

  root
  package.json
    packages
      package1
        package.json
      package2
        package.json

### command line

`>>> yarn why pseudomap 1.0.2`

### outpout

```
pseudomap@^1.0.1 (1.0.2)
     lru-cache@^3.2.0 (3.2.0)
          editorconfig@^0.13.2 (0.13.3)
               js-beautify@^1.5.1 (1.7.4) - devDependency of package1
pseudomap@^1.0.2 (1.0.2)
     lru-cache@^4.0.1 (4.1.1)
          cross-spawn@^3.0.0 (3.0.1)
               node-sass@^4.5.3 (4.6.1) - dependency of package1
          cross-spawn@^5.0.1 (5.1.0)
               execa@^0.7.0 (0.7.0)
                    os-locale@^2.0.0 (2.1.0)
                         yargs@11.0.0 (11.0.0)
                              webpack-dev-server@3.1.4 (3.1.4) - devDependency of package1 and package2
                         yargs@^10.0.3 (10.0.3)
                              jest-cli@^22.4.2 (22.4.4)
                                   jest@22.4.2 (22.4.2) - dependency of package1 and devDependency of package2
                              jest-runtime@^22.4.4 (22.4.4)
                                   jest-cli@^22.4.2 (22.4.4)
                                        jest@22.4.2 (22.4.2) - dependency of package1 and devDependency of package2
                                   jest-runner@^22.4.4 (22.4.4)
                                        jest-cli@^22.4.2 (22.4.4)
                                             jest@22.4.2 (22.4.2) - dependency of package1 and devDependency of package2
               execa@^0.8.0 (0.8.0) - dependency of package2
                    lerna@^2.2.0 (2.5.1) - dependency of the main package.json
                    lint-staged@^4.2.1 (4.3.0) - dependency of the main package.json
```

# How We Teach This

The command `yarn why` is already known and I believe this design fits better the
expectations than the current implementation.

The documentation will of course be updated but will not need a reorganization.

If the `yarn why` command is used to teach yarn to new users, then the
teaching materials will need to be updated.

If this design solves some problems better than current tools and the current solution
is documented, then the documentation of this problem solution will need to be updated
to use the new command.

Advanced users will benefit from being introduced to this feature.

# Drawbacks

Some people may have implemented some scripts on top of `yarn why`, these scripts
will be broken.

The command `yarn why` will become more verbose and it may confuse people that are
used to the current command.

The tool will need to use both `yarn.lock` and `package.json` files, it will be buggy if
these are out of sync because of manual changes.

# Alternatives

This tool could be implemented as a separate package.

An alternative is to run recursively `yarn why`.

Every users write their own script for their specific problem.

# Unresolved questions

Do we show ALL the dependency chains leading to the desired package? Do we merge certain
paths together when they share most of their packages? If so how will this look like?

Should we have an option to skip devDependencies?

What happens if someone manually changes package.json and run the `yarn why` command without
running `yarn` first?

How does this command works when ran before a merge conflict in yarn.lock is resolved?

What is the source of truth? The `yarn.lock` and `package.json` files or the actually installed packages?

How to treat circular dependencies?
