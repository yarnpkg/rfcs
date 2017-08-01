- Start Date: 2017-07-24
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Ability to opt-in install scripts for just select modules

# Motivation

Install scripts can be problematic from a security perspective, as eg. noted in https://www.kb.cert.org/vuls/id/319816 and [acknowledged by npm](http://blog.npmjs.org/post/141702881055/package-install-scripts-vulnerability). It's also not unheard of that install scripts eg. messes up a recursive delete and deletes unintended data, like eg. [Adobe accidentally did](https://arstechnica.com/apple/2016/02/warning-bug-in-adobe-creative-cloud-deletes-mac-user-data-without-warning/).

Blocking install scripts all together with `ignore-script` though, as is current the only option, also blocks all legitimate use cases, such as modules that needs to compile some code.

From a security perspective it would become more manageable if one could opt-in specific modules and have only the install scripts of those modules be run.

By opting-in one can vet every module before running its code and decide on a module by module basis whether to allow it or not. Vetting and opting-in individual modules allows for the full functionality of compiled modules while keeping the security exposure of a project to a minimal.

To enable tools and interfaces to be tried out and patterns to emerge, such that may later be brought into yarn, the proposed design is kept to the minimal necessary core where support is needed within yarn itself.

# Detailed design

For this to work, some things are needed:

1. A way to opt-in the install scripts of a module
2. A way to enforce such a opt-in list

## 1. Opt-in the install scripts of a module

A new flag, `--allow-script`, on `yarn add` and `yarn upgrade` would add an `allowScript` key to that modules entry in `yarn.lock` that tells that the script has its install scripts opted-in to.

## 2. Enforce the opt-ins

A new option that's mimicks `ignore-scripts`, but that allows opted-in scripts (`only-allowed-scripts`) would be a good first step. It would apply both as a global configuration and as a flag on `yarn install`.

Later on it would be good to make it configurable on a project level as well – through `package.json`, `yarn.lock` or somewhere else.

# How We Teach This

As this proposal stands now this functionality would be a fully opt in and not something that a first time user would have to know about.

If it in the future becomes possible to configure a project to have the opt-ins imposed on a project level, then it needs to be communicated to new contributors of such a project that such opt-ins are enforced for them so that they when they add or update new dependencies are aware that the opt-ins may need to be updated as well for the new dependencies to fully functional.

As this proposal stands now only the options would have to be communicated – how one opts-in to something and how one choses to enforce those opt-ins. Not that different from how the functionality of `ignore-scripts` is communicated today.

# Drawbacks

Having a opt-in complicates the list of dependencies and makes the `yarn.lock` more complex.

It also makes the installation process more complex as it becoming harder to know if the install scripts of a module has been run or not as its no longer an all or nothing as is the case of `ignore-scripts`. As it can be somewhat hard today to know which packages that use install scripts and which don't it may to hard to know why a package no longer works as expected. This could be mitigated by clearly communicating whenever an install scripts has been ignored due to not being opted-in to.

As it would become a yarn-only feature it would also leave any users of npm (and of older yarn versions) possibly more vulnerable as less care would be taken to vet the install scripts of modules not opted-in to. If these however are already using `ignore-scripts`, then things would remain as secure for them.

# Alternatives

## Private registry

In some previous discussions on similar matters, https://github.com/node-forward/discussions/issues/29#issuecomment-70438448, it was noted that private registries can be used to achieve a similar vetting of modules in there and one could probably also modify packages in such a private registry to remove any install scripts one would prefer not to run. Such private registries are though not easily accessible to people + adds extra complexity to a setup that makes it harder for people to contribute and to eg. opt-in new modules through pull requests and such.

# Unresolved questions

## Where to store opt-ins

Should it be in the `yarn.lock` or somewhere else, like in the `package.json`?

## Project specific configuration

Should enforcement of opt-ins solely happen through flag and global configuration or should a project be able to enforce it at a project level as well, embedding the enforcement in eg. the `yarn.lock` or `package.json` so that everyone else who installs the project with yarn automatically gets the opt-ins enforced?

## How to handle the need of opting-in install scripts of a sub-dependency

A module like [bunyan](https://github.com/trentm/node-bunyan) which has a dependency that needs compilation, ([dtrace-provider](https://github.com/chrisa/node-dtrace-provider)), would have to get that dependency opted-in to rather than to have itself become opted into. As one only do `yarn add` on the top level dependency there would be no way of opting-into such a sub-dependency through any existing command so either a new command for opting-into a specific dependency and/or an extension of `upgrade-interactive` as well a new `add-interactive` would be needed to allow opting-into such sub-dependencies.,

An extended `upgrade-interactive` and a new `add-interactive` would make it easy to opt-into sub-dependencies and to ensure that updates to opted-into modules are vetted before they get opted-into. An `add-interactive` could point out any non-opted-into sub-dependency that wants to run an install script and ask whether that should be allowed and opted-into or whether that should be ignored. If ignored, then that should probably be explicitly saved as well so one don't get bombarded with the same question again and again. Like saving an explicit `allowScript: false` to `yarn.lock` for that module.

By keeping the design of this out of the initial proposal the hope is that experimentation with tools and interfaces for handling these cases can emerge outside of yarn and that yarn later, if the need remains, can bring the best of such experiments into core.
