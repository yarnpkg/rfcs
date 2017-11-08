- Start Date: 2017-11-04
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Adding a mechanism "nohoist" to allow monorepo projects, who utilize yarn's [workspaces](https://yarnpkg.com/en/docs/workspaces), to opt-out the default hoisting behavior.

# Motivation

See issue [#3882](https://github.com/yarnpkg/yarn/issues/3882) for why developers are asking for `nohoist` features. To summarize:

## use cases:
1. a monorepo project with at least one workspace depends on react-native.
    * this is the use case for nohoisting specific direct-dependency from a workspace.
1. tools like grunt assumes directory structure.
    * this could require nohoist both shallow and deep dependency trees
1. some projects may prefer no-hoisting for whatever reason ;-)

## nohoist address a common issue in hoisted monorepo project

`nohoist` is not a new concept, [lerna](https://github.com/lerna/lerna#--nohoist-glob) has provided a similar feature, an indicator that this is a common issue many monorepo projects have to deal with.

# Detailed design

## principle
- hoist as much as we can
- nohoist is an exception/workaround, therefore favor 
    - explicitness over convenience
    - small scale over large scale

These principles lead us to the "**shallow nohoist**" implementation: _nohoist only apply to the package explicitly specified; all of its dependency, unless also on the nohoist list, will be hoisted by default._

## divide the problem 

nohoist is meant to address the hoist side-effect, thus break down by scale, from hoisting all to none:
```
hoist-all(1) --> nohoist-direct-dep(2) --> nohoist-deep-dep(3) --> hoist-none(4)
```
1. hoist-all: hoisting everything, the current behavior of yarn workspaces.
2. nohoist-direct-dep: opt-out hoisting for direct dependency only, use-case #1
3. nohoist-deep-dep: opt-out hoisting even for dependencies' dependencies... Maybe use-case #2
4. hoist-none: a.k.a nohoist all, opt-out hoisting all together: use-case #3.

yarn currently support (1), this RFC proposed solution to handle (2), (3), (4).

## configuration
We need a place to specify "nohoist" packages for the given workspace(s). We can expand the existing [workspaces](https://yarnpkg.com/en/docs/workspaces) config:
```
export type WorkspacesConfig = {
  packages?: Array<string>,
  nohoist?: Array<string>,
};

export type Manifest = {
    ...
    workspaces?: Array<string> | WorkspacesConfig
}
```
note: the config will be backward compatible.

### nohoist list inheritance
nohoist config is inherited, i.e. if nohoist is specified in the root package.json, all workspaces will inherit the nohoist list, likewise all the dependencies of the workspace will inherit the nohoist list. This concept is simple and useful not only in reasoning nohoist from root to workspaces, it can be extended downwards to allow packages provide better monorepo support as well as easily handling deep-dependency as direct-dependency. 

### nohoist list naming
lerna uses glob pattern to specify nohoist package, however based on the design principle, I argue it is probably better not to since nohoist should be "rare", therefore probably doesn't justify for extra matching complexity and risk. 

However, it is not feasible to ask developers mark off every package if they just want to opt-out hoisting all together, therefore a simple keyword like "\_all\_" should be introduced to handle use-case-3 easily. For example the following config will basically turn off nohoist for the whole monorepo project:
```
// root packages.json
workspaces = {
    packages: ['workspace-a', 'workspace-b'],
    nohoist: ['_all_']
}
```

### workspace specific nohoist

As some commented in [#3882](https://github.com/yarnpkg/yarn/issues/3882), it seems reasonable to customize hoisting by workspace. This can be easily achieved to either allow nohoist config in workspace's package.json (option-1) or make root package's config to be workspace-aware(option-2).  

for example, to only disable hoisting for workspace-a:

**option-1: in workspace's package.json**: 
```
// workspace-a package.json
workspaces = {
    nohoist: ['_all_']
}
```
**option-2: in root package.json**:
```
// root package.json
workspaces = {
    packages: ['workspace-1', 'workspace-b']
    nohoist: {'workspace-1': ['_all_']}
}
```
IMHO, option-1 is more intuitive, elegant and extensible. However, workspaces might not be private packages. @bestander has mentioned the concern about non-yarn use cases, I added more thoughts in [discussion #1](#discussion-1). If this turns out to be a real issue, we could always fallback to option-2. 

### exceptions
The following packages will ignore the nohoist instruction:
- workspaces
- linked packages

Basically the symlink-packages will always be hoisted because they are not copied hence can't honor the nohoist instruction.

for example: 
```
// workspace-2 package.json
dependencies: {
    "a": "1.0.0",
    "workspace-1": "1.0.0"
}
workspaces: {
    "nohoist": ["_all_"]
}
```
workspace-1 will ignore "nohoist" specified by workspace-2, so the final file structure will look like this:
```
root/node_modules/
    workspace-1
    workspace-2/node_modules/a
```

note: I have considered alternatively to create  `workspace-2/node_modules/workspace-1` as a symlink to workspace-1, but then realized if there are multiple workspaces referencing workspace-1 with conflicting nohoist rules, the shared workspace-1 can't satisfy all of them and will appear "wrong" if you just follow the symlinks... Therefore, I think it might be easier to just mark these linked-packages to be not-nohoist-able, i.e. they will always be hoisted regardless of the nohoist config.

## nohoist logic outline

all of the nohoist logic (minus the config) is implemented in `src/package-hoist.js`:

1. added a new property `HoistManifest._nohoistList` to record nohoist package list. 
1. during `_seed()`, the new HoistManifest populate the `_nohoistList` from its own package.json (if any) + parent's. 
1. during the `PackageHoister.hoist()`, the logic only need to be diverged on how to find the "hoisting" point: nohoist packages will only be hoisted to break circular reference while the other packages go through the full `getNewParts()`. The rest of the hoist logic can remain the same. 
1. if the package is hoisted to a new location, the _nohoistList of the new hoisted manifest is reset so it will look like any hoisted package. (I am open on this, see [discussion-2](#discussion-2))

_The actual code change is surprisingly minimal... it took me longer to write this proposal than making the code change_ ;-)

# How We Teach This

as mentioned earlier, `nohoist` is not a new concept, monorepo projects that use and understand how package hoisting work should be easy to pick up the concept of not-hosting. We could introduce a few concrete nohoist examples (such as the following) in the existing [workspaces](https://yarnpkg.com/en/docs/workspaces) document as a starting point, expand as needed.

## examples
a monorepo project with 2 workspaces: 'workspace-a', 'workspace-b':
```
// root package.json
workspaces = {
    packages: ['workspace-a', 'workspace-b']
}
```
### example 1: not hoisting react-active for workspace-a
```
// workspace-a package.json
workspaces = {
    nohoist: ['react-native']
}
```
the file structure will end up like this:
```
    monorepo
      |- node_modules/
        |- workspace-b
        |- (... all workspace-b's dependencies)
        |- (... all workspace-a's dependencies, except react-native)
        |- (... all react-native's dependencies, such as core-js)
        |- workspace-a
            |- node_modules/
              |- react-native
                |- node_modules/
                   (empty)

```
### example 2: disable hoisting react-native and one of its dependency, "core-js":
```
// workspace-a package.json
workspaces = {
    nohoist: ['react-native', 'core-js']
}
```
the file structure will look like this:
```
    monorepo
      |- node_modules/
        |- workspace-b
        |- (... all workspace-b's dependencies)
        |- (... all react-native's dependencies, minus core-js)
        |- (... all workspace-a's dependencies, except react-native)
        |- workspace-a
            |- node_modules/
              |- react-native
                |- node_modules/
                   |- core-js
                     |- node_modules/
                        (empty)

```
_note: only the package explicitly called out in nohoist list will be excluded from hoisting. Therefore, in order to exclude core-js, react-native has to be included in the nohoist list._

### example 3: disable hoisting for workspace-a
```
// workspace-a package.json
workspaces = {
    nohoist: ['_all_']
}
```
the file structure will look like this:

```
    monorepo
      |- node_modules/
        |- workspace-b
        |- (... all workspace-b's dependencies)
        |- workspace-a
            |- node_modules/
              |- (... all workspace-a's dependencies...)

```
### example 4: disable hoisting for the whole monorepo project
```
// root package.json
workspaces = {
    workspaces = ['workspace-a', 'workspace-b'],
    nohoist: ['_all_']
}
```
the file structure will look like this:
```
    monorepo
      |- node_modules/
        |- workspace-b
            |- node_modules/
              |- (... all workspace-b's dependencies...)
        |- workspace-a
            |- node_modules/
              |- (... all workspace-a's dependencies...)

```

# Drawbacks

* Why should we *not* do this?

    - We could argue the current implementation is simple, consistent and correct, therefore, should not introduce nohoist to muddy the water. 
    - The violating packages, such as react-native, should be corrected to use package manager properly instead of assuming module locations. 
    - If developers prefer no-hoist, they could just not use "workspaces" or write their own custom scripts to deal with project specific needs.

* Tradeoffs
    
    This is a classic simplicity vs usability. yarn can be simple and beautiful but if developers find it hard to use in practice, they will not use it. 

# Alternatives

* What other designs have been considered?

    - There are a few [high level options](https://github.com/yarnpkg/yarn/issues/3882#issuecomment-338478889) discussed in [#3882](https://github.com/yarnpkg/yarn/issues/3882). 
    - More implementation alternatives are also marked as discussion in their corresponding sections above.  
    - we could choose not support nohoist-deep-dep(3), as use case (#2) is not very clear to me if we need this or what the expectation is. However, with inherited nohoistList, we get deep-nohoist pretty much for free so decided to just leave it in. We could either not document it publicly or leave it as it is and refine later when needed.

# Unresolved questions

## discussion-1
**is allowing nohoist in public package bad?**

As mentioned earlier, nohoist only impact yarn hoisting, for other package managers, it's a no-op and should be able to safely ignored... maybe I am missing some problematic use cases, here is what I got:

Let's say a user project A has a dependency on a public package B, which has a dependency on 'react-native' that will need to be excluded from hoist otherwise it won't work. 

||B has workspaces.nohoist=react-native|B didn't have workspaces.nohoist|
|--|--|--|
|A doesn't use yarn or workspace|no impact|no impact|
|A uses yarn workspace|react-native will be excluded from hoisting without A doing anything|A needs to discover then put both B and react-native in its own workspaces.nohoist|
|A uses yarn workspaces but excluded \_all\_|no impact, none will be hoisted|no impact, none will be hoisted|
|A uses yarn workspaces but do not want to exclude react-native from hoisting (why?)|conflict! but the public package B is right and react-native should be excluded.|no conflict but B will most likely failed|

In short, having unnecessary nohoist packages might cause inefficiency, but missing a necessary nohoist will lead to compile/execution error.

## discussion-2
**should the new hoisted package retain nohoist list?**
example:
```
workspace-a -> A -> B
```
if developer added 'B' in the workspace-a's nohoist config:
- package A will inherit nohist list ('B') from workspace-a
- package A will be hoisted since it's not on the nohoist list ('B'), 
- option-1: if we remove the nohoist list from A at this point, package B will be hoisted as well, the final graph will look like:
```
workspace-a
A
B
```
- option-2: if we don't remove the nohoist list from A, package B will stay under A as it is in the nohoist list:
```
workspace-a
A -> B
```
IMHO, option-2 is more confusing than option-1. We can reason the nohoist inheritance is based on the dependency tree, once the tree is broken (by hoisting), so should the nohoist list inheritance. But I am open for better arguments.

## unit tests failed on mac
this is not related to nohoist issue specifically, but made submitting PR much more difficult and time consuming. Many async integration tests failed on my laptop (mac) for even a fresh clone. Is this a known issue? any suggestion or workaround?
