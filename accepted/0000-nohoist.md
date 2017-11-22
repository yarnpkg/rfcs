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

## Summary: what does nohoist really do
At the end of the day, nohoist is just a mechanism to determine a different 'hoist' point. By default, **all** packages in the workspaces are hoisted to the **root**, except for version conflicts (similar to [npm resolution](https://docs.npmjs.com/how-npm-works/npm3)). Nohoist provided a mechanism to custom this algorithm and determine:
1. what packages should be the exception of the rule ?

    This is majority of the work for the rest of the RFC, we will describe how we can use the glob pattern to match the package in the dependency graph, like a virtual file system.

2. where to place them?
    There are general 2 options:
    - option-1: under the module that reference it. 
    - option-2: under the workspace that reference it.

    I have tried both implementations and realized why most package manager moving to the hoisting model... By leaving the packages under the referencing module (option-1), I quickly bumped into the scalability issue (ran out of ram during `yarn install`) for larger project due to high number of (duplicated) modules. Not to mention the complexity in resolving the [modules-in-between](#module-in-between) issue...

    After examined how react-native project worked outside of the workspace environment (both yarn and npm did host everything to the \$cwd/node\_modules), I realized that developers are already trained to go **down** the node\_module tree, but not every one has followed the [node module resolution algorithm](https://nodejs.org/api/modules.html#modules_loading_from_node_modules_folders) to go **up** from \$cwd. It is generally not a problem, unless you are in the monorepo project where modules can reside above the workspace referencing them...

    This drives me to settle on option-2 by simply hoisting the nohoist modules in the workspace's node\_modules (instead of the project's root) where they can still share the benefit of hoisting and greatly reduce the complexity in implementation and education. This essentially made the workspace/node_modules looked like a stand alone project, for the given nohoist-modules. Therefore, if a package can work in a stand alone project, it should be able to work under workspaces.

## principle
- hoist as much as we can
- nohoist is an exception/workaround, therefore favor 
    - explicitness over convenience
    - small scale over large scale

These principles lead us to the following implementation: _nohoist only apply to the package explicitly specified; all of its dependency, unless also matched the nohoist list, will be hoisted by default._

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
considering the dependency as a virtual tree, the nohoist rule goes down the branch from where it is specified. For example, if nohoist is specified in the root package.json, all workspaces will inherit the nohoist list, likewise all the dependencies of the workspace will inherit the nohoist list. However if the nohoist is specified in workspace-1, the neighboring workspace-2 will not be impacted. 

### nohoist list matching
Like the current workspaces.packages configuraton, we will use the same glob pattern match to locate nohoist packages. [minimatch](https://github.com/isaacs/minimatch) (the matching library used in yarn) supports many glob patterns, the most common ones for nohoist are probably the simple '*' and '**' patterns:

For example the following config will basically turn off nohoist for the whole monorepo project:
```
// root packages.json
workspaces = {
    packages: ['workspace-a', 'workspace-b'],
    nohoist: ['**']
}
```

But why '\*\*' ([globstar](http://www.linuxjournal.com/content/globstar-new-bash-globbing-option)) and not '\*'? By using globstar we can support both shallow ('\*' matches only 1 level) and deep matches (globstar matches all levels down).

Another example, the `nohoist: ['workspace-1/**']` will only disable workspace-1's hoisting. We get workspace specific config through glob pattern, nice...

To make react-native workspace work, I added `nohoist: ['**/react-native/**']` in the root package.json before installing react-native to tell yarn don't hoist react-native and all of its dependency, wherever they are. 

Note: right now I am leaning toward file path like format, this works intuitively with glob pattern matching. However the rest of the system seem to use '#' as the separator, as seen in HoistManifest.key. Please see [here](#dependency-tree-representation) for more discussion.

### workspace specific nohoist
Workspace, or any package in that matter, can also specify its own nohoist rule just like in root. This might be useful for 3rd-party package to specify nohoist packages to assist monorepo project adoption. The consumers of these packages will automatically pick up the nohoist rule without adding its own nohoist.

for example, if react-native's package.json has `nohoist: ['**']`, the workspace that referencing react-native will not need any special config to get the same effect as `**/react-native/**`.

### How about links and workspaces dependencies?
by not leaving the packages at where it is referenced, we can now handle links and workspace dependency just like any package. For example, if workspace-2 depends on workspace-1 and would need to have workspace-1 traversable from workspace-2/node_modules: `nohoist: ['workspace-1']`. The tree will be as you expected:
```
_project_/node_modules
    |- workspace-2/node_modules
        |- workspace-2 (symlink)
    |- workspace-1
```
and if workspace-1 has hoisted everything to the _project_ root, workspace-2 might still not able to access workspace-1's dependencies, it could then 
```
specify: `nohoist: ['workspace-1/**']`:
_project_/node_modules
    |- workspace-2/node_modules
        |- workspace-2 (symlink)
        |- all workspace-1's dependent packages (copy)
    |- workspace-1
``` 
This allow both workspaces to have their own hoisting rules. Same thing should apply to linked modules.

## nohoist logic outline

all of the nohoist logic (minus the config) is implemented in `src/package-hoist.js`:

1. added the following property in HoistManifest: 
    - `nohoistList` to record nohoist pattern. 
    - `isNohoist` to record nohoist state. 
    - `originalParentPath` to record the dependent tree before hoisting. Nohoist rule is evaluated against this, not the after-hoisting path (key). 
1. during `PackageHoister._seed()`, populate the new properties above by examining the parent and its own package.json.
1. during the `PackageHoister.getNewParts()`, the logic only need to be added on deciding what is the highest hoisting point. By default, the highest hoisting point is always root(0), unless the package is marked as nohoist, in which case the highest hoisting point is its workspace (1). 

notes:
- doesn't matter where the packages end up, they will always have access to the dependency tree before hoisting and the nohoist rules. This will come handy when we need to investigate with `yarn why`

_The actual code change is surprisingly minimal... it took me longer to write this proposal than making the code change_ ;-)

## Incompatible assumptions  
During the testing, I encountered a few hidden assumptions that need to be changed:
- `add.js`: added package always go in with the root, regardless \$cwd.  
    This is fine in everything-hoist-to-root world, but not true with nohoist package. When calling `yarn add` from the workspace to add a nohoist package, even with nohoist specified in the package.json, the new package will still be placed under root because it is being 'seeded' explicitly with root. I had to modify the add.js to make it behave like a regular install, i.e no manual seeding, just add the new package into the in-memory Manifest pool and let _install.js_ do its normal thing. 
    
- `why.js`: only reported the first encountered package, assumed there is only 1 in the project...
    This seems like a bug, a package can indeed exist in multiple places today, due to version conflict. One would think _why.js_ should report all of them. With nohoist, this is even more critical. 
    
    Also the current _why.js_ reported dependency based on the "post hoist" structure, which might not tell the full tree at once. Consider a/b/c, all of them has been hoisted to the root. if you ask why 'c', it will say 'b', and you have to ask 'b' again to get 'a'. Giving we have the originalPath now, we can report 'c' is from 'a/b' in one go. I fixed these issues to help me debug and hopefully will help others too.

 As one can see, some assumptions like above, maybe fine before, now will need to be adapted after nohoist. The list is far from complete. I have only addressed those stood in my way of getting the test case 1 (react-native workspace) working. Figure it is better to get the basic features out there earlier than later, so we can be sure the core implementation is sound before piling up more changes...

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
### example 1: disable hoisting for react-active in workspace-a
```
// workspace-a package.json
workspaces = {
    nohoist: ['react-native']
}
```
the file structure will end up like this:
```
    monorepo/node_modules/
        |- (... all workspace-b's dependencies)
        |- (... all workspace-a's dependencies, except react-native)
        |- (... all react-native's dependencies, such as core-js)
        |- workspace-b
        |- workspace-a/node_modules/
            |- react-native/node_modules/ (empty)

```
### example 2: disable hoisting for react-native and its dependencies in workspace-1:
```
// workspace-a package.json
workspaces = {
    nohoist: ['react-native', 'react-native/**']
}
```
the file structure will look like this:
```
    monorepo/node_modules/
        |- (... all workspace-b's dependencies)
        |- (... all workspace-a's dependencies, except react-native and its dependencies)
        |- workspace-b
        |- workspace-a/node_modules/
            |- react-native/node_modules/
            |- (... all react-native's dependencies)

```

### example 3: disable hoisting for workspace-a
```
// workspace-a package.json
workspaces = {
    nohoist: ['**']
}
```
the file structure will look like this:

```
    monorepo/node_modules/
        |- (... all workspace-b's dependencies)
        |- workspace-b
        |- workspace-a/node_modules/
            |- (... all workspace-a's dependencies...)

```
### example 4: disable hoisting for the whole monorepo project
```
// root package.json
workspaces = {
    workspaces = ['workspace-a', 'workspace-b'],
    nohoist: ['**']
}
```
the file structure will look like this:
```
    monorepo/node_modules/
        |- workspace-b/node_modules/
           |- (... all workspace-b's dependencies...)
        |- workspace-a/node_modules/
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
    - Some implementation alternatives are discussed inline above. 
    - The rest is documented in the following section.

# Unresolved questions

## dependency tree representation
in order to utilize glob paradigm, I used the file-path representation for originalPath (= the before-hoist dependency tree). Currently yarn uses '#' as the separator to describe the after-hoist tree, such as in key, parentKeys etc. Not sure if there is a specific reason with '#'. It is probably better to be consistent at least for external reporting purpose. 

Here are the reasons I choose path like syntax:
- minimatch apply globstar only for path like strings: 'a#b#c' doesn't match '**', 'a/b/c' does.
- visually, I find 'a/b/c' a lot easier to read than 'a#b#c'
- '/' is a pretty common way to specify tree structure in school and publications. 

None of them is hard to overcome, we just need to decide a format then adapt the rest. 
 
## is allowing nohoist in public package bad?

As mentioned earlier, nohoist only impact yarn hoisting, for other package managers, it's a no-op and should be able to safely ignored... maybe I am missing some problematic use cases, here is what I got:

Let's say a user project A has a dependency on a public package B, which has a dependency on 'react-native' that will need to be excluded from hoist otherwise it won't work. 

||B has workspaces.nohoist=react-native|B didn't have workspaces.nohoist|
|--|--|--|
|A doesn't use yarn or workspace|no impact|no impact|
|A uses yarn workspace|react-native will be excluded from hoisting without A doing anything|A needs to discover then put both B and react-native in its own workspaces.nohoist|
|A uses yarn workspaces but excluded all packages from hoisting|no impact, none will be hoisted|no impact, none will be hoisted|
|A uses yarn workspaces but do not want to exclude react-native from hoisting (why?)|conflict! but the public package B is right and react-native should be excluded.|no conflict but B will most likely failed|

In short, having unnecessary nohoist packages might cause inefficiency, but missing a necessary nohoist will lead to compile/execution error.

[update 11/13/17]
After further discussion, I agree that to have an option guarding nohoist from public package is a safer approach so users can turn off nohoist along with workspaces feature. It is not clear to me if we need a new flag to handle nohoist opt-out separately from workspaces. A nohoist without workspaces is meaningless because nohoist is essentially part of the workspace hoisting scheme. If we do have use cases to override specific public package's nohoist, a generic flag might not be sufficient... Therefore, suggest we hold off on any complex addition until seeing a concrete use case, mean while just use the existing `workspaces-experimental` to safe guard workspaces as a whole.

## unit tests failed on mac
this is not related to nohoist issue specifically, but made submitting PR much more difficult and time consuming. Many async integration tests failed on my laptop (mac) for even a fresh clone. Is this a known issue? any suggestion or workaround?
