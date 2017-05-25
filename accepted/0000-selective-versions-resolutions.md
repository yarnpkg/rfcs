- Start Date: 2017-05-21
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Allow to select a nested dependency version via the `resolutions` field of
the `package.json` file.

# Motivation

The motivation was initially discussed in
[yarnpkg/yarn#2763](https://github.com/yarnpkg/yarn/issues/2763).

Basically, the problem with the current behaviour of yarn is that it is
not possible to force the use of a particular version for a nested dependency.

## Example

For example, given the following content in the `package.json`:
```json
 "devDependencies": {
    "@angular/cli": "1.0.3",
    "typescript": "2.3.2"
  }
```

The `yarn.lock` file will contain:
```
"typescript@>=2.0.0 <2.3.0":
  version "2.2.2"
  resolved "https://registry.yarnpkg.com/typescript/-/typescript-2.2.2.tgz#606022508479b55ffa368b58fee963a03dfd7b0c"

typescript@2.3.2:
  version "2.3.2"
  resolved "https://registry.yarnpkg.com/typescript/-/typescript-2.3.2.tgz#f0f045e196f69a72f06b25fd3bd39d01c3ce9984"
```

Also, there will be:
- `typescript@2.3.2` in `node_modules/typescript`
- `typescript@2.2.2` in `node_modules/@angular/cli/node_modules`.

## Problem

In this context, it is impossible to force the use of `typescript@2.3.2` for
the whole project (except by flattening the whole project, which we don't want).

It makes sense for typescript as the user intent is clearly to use typescript
2.3.2 for compiling all its project, and with the current behaviour, the angular
CLI (responsible of compiling `.ts` files) will simply use the 2.2.2 version
from its `node_modules`.

Similarly, even using such a content for `package.json`:
```json
 "devDependencies": {
    "@angular/cli": "1.0.3"
  }
```

The need could arise for forcing the use of `typescript@2.3.2` (or
`typescript@2.1.0` for that matter).

## Why?

In these example, the need does not seem very important (the user could maybe 
use `typescript@2.2.2` or ask the `@angular/cli` dev team to relax its
constraints on typescript), but there could be cases where a nested dependency
introduces a bug and the project developer would want to set a specific
version for it (see for example this
[comment](https://github.com/yarnpkg/yarn/issues/2763#issuecomment-302682844)).

## Related scenario (out of scope of this document)

An extension of this motivation is also the potential need for mapping nested
dependencies to others. For example a project developer could want to map
`typescript@>=2.0.0 <2.3.0` to `my-typescript-fork@2.0.0`.

See alternatives solutions below also.

# Detailed design

The proposed solution is to make the `resolutions` field of the `package.json`
file to be considered all the time and on a per-package basis (instead of
only when the `--flat` parameter is used).

When a nested dependency is being resolved by yarn, if the `resolutions` field
contains a specification for this package, then it will be used instead.

Special attention to the specific nature of package management in the npm
ecosystem is given in this RFC: indeed, it is not unusual to have the same
package being present as a nested dependency of multiple packages with
different versions. It is thus possible within the `resolutions` field to
express versions either for the whole dependency tree or only for a subset of
it, using a syntax relying on glob patterns.

Most of the examples are given with exact dependencies, but note that using a
non-exact specification in the `resolutions` field should be accepted and
resolved by yarn like it usually does. This subject is discussed below also.

Any potentially counter-intuitive situation will result in a warning being
issued. This subject is discussed at the end of this section.

## Examples

We the following packages and their dependencies:
```
package-a@1.0.0 and 2.0.0
 |_ package-d1@1.0.0
     |_ package-d2@1.0.0

package-b@1.0.0
 |_ package-d1@2.0.0

package-c@1.0.0
 |_ package-a@2.0.0
     |_ ...
```

With:
```json
 "devDependencies": {
    "package-a": "1.0.0",
    "package-b": "1.0.0"
  },
  "resolutions": {
    "**/package-d1": "3.0.0"
  }
```

yarn will use `package-d1@3.0.0` for every nested dependency to `package-d1`
and will behave as expected with respect to the `node_modules` folder by not
duplicating the `package-d1` installation.

With:
```json
 "devDependencies": {
    "package-a": "1.0.0",
    "package-b": "1.0.0"
  },
  "resolutions": {
    "package-a/package-d1": "3.0.0"
  }
```

yarn will use `package-d1@3.0.0` oly for `package-a` and `package-b` will
still have `package-d1@2.0.0` in its own `node_modules`.

With:
```json
 "devDependencies": {
    "package-a": "1.0.0",
    "package-c": "1.0.0"
  },
  "resolutions": {
    "**/package-a": "3.0.0"
  }
```

`package-a` will still be resolved to `1.0.0`, but `package-c` will have
`package-a@3.0.0` in its own `node_modules`.

With:
```json
 "devDependencies": {
    "package-a": "1.0.0",
    "package-c": "1.0.0"
  },
  "resolutions": {
    "package-a": "3.0.0"
  }
```

yarn will do nothing (see below why).

With:
```json
 "devDependencies": {
    "package-a": "1.0.0",
    "package-c": "1.0.0"
  },
  "resolutions": {
    "**/package-a/package-d1": "3.0.0"
  }
```

yarn will use `package-d1@3.0.0` both for `package-a` and the nested
dependency `package-a` of `package-c`.

## Resolutions

Each sub-field of the `resolutions` field is called a *resolution*.
It is a JSON field expressed by two strings: the package designation on the
left and a version specification on the right.

### Package designation

A *resolution* contains on the left-hand side a glob pattern applied to
the dependency tree (and not to the `node_modules` directory tree, since the
latter is the result of yarn resolution being influenced by the *resolution*).

- `a/b` denotes the directly nested dependency `b` of the project's
dependency `a`.
- `**/a/b` denotes the directly nested dependency `b`
of all the dependencies and nested dependencies `a` of the project.
- `a` denotes the project's dependency `a` (see below for a discussion on
the matter of specifying a *resolution* for one of the project dependency).
- `a/**/b` denotes all the nested dependencies `b` of the project's
dependency `a`.
- `**/a` denotes all the nested dependencies `a` of the project.
- `**/a-*` denotes all the nested dependencies of the project whose
name starts with `a-`.
- `**` denotes all the nested dependencies of the project (a bad idea mostly,
as well as all other designations ending with `**`).

### Version specification

A *resolution* contains on the right-hand side a version specification
interpreted via the `semver` package as usually done in yarn.

## Relation to non-nested dependencies

The `devDependencies` and `dependencies` fields always take precedence over the
`resolutions` field: if the user defines explicitly a dependency there,
it means that he wants that version, even if it's specified with a non-exact
specification. So the `resolutions` field only applies to nested-dependencies.
Nevertheless, in case of incompatibility between the specification of a
non-nested dependency version and a *resolution*, a warning is issued.

## Relation to the `--flat` option

The `--flat` option becomes thus a way to populate the resolutions field for
the whole project, as it already does (using a package designation in the
form of `**/package-name`).
And the `resolutions` field is always considered by yarn, even when `--flat` is
not specified.

Incidently, this resolves this strange situation when two developers would be
working on the same project, and one is using `--flat` while the other is not,
and they would get different `node_modules` contents because of that.

## `yarn.lock`

This design implies that it is possible to have for a given version
specification (e.g., `>=2.0.0 <2.3.0`) a resolved version that is incompatible
with it (e.g., `2.3.2`). It is acceptable as long as it is explicitly
asked by the user via a *resolution*.

It is currently the case that such situation would make yarn unhappy and
provoke the modification of the `yarn.lock` (see 
[yarnpkg/yarn#3420](https://github.com/yarnpkg/yarn/issues/3420)).

This feature would remove the need for this behaviour of yarn.

## Non-exact version specifications

If there is a non-exact specifications in the `resolutions` field, the rule is
the same: the `resolutions` field takes precedence over the specification in a
nested dependency.

In case the `resolutions` field is broader than the nested dependency
specification, then a warning can be issued. This happens if the the exact
version resolved by yarn based on the `resolutions` specification is
incompatible with the nested dependency specification.

For example, if `@angular/cli` depends on `typescript@>=2.0.0 <2.3.0` and the
`resolutions` field contains `typescript@>=2.0.0 <2.4.0`, then if the latest
available version for typescript is `2.2.2`, no warning is issued, and if the
latest available version for typescript is `2.3.2` then a warning is issued.

The rational behind that is that since the `yarn.lock` file is only modified
by the user (via yarn commands), then a warning will always be issued before
such a situation happens and is written to the `yarn.lock` file.

## Warnings in logs

yarn should warn about the following situations:
1. Unused resolutions.

2. Incompatible resolutions (see also above the sections about `yarn.lock`
and about broadening non-exact specifications).
Basically, an incompatible resolution is used because a package does not
correctly express its dependencies. In an ideal world, the package should
be fixed at one point or another and the resolution should be removed.
In that sense, incompatible resolutions should always be warned about.
Furthermore, an incompatible resolution is a potential for unwanted behaviour
and should thus never be ignored by the user.

# How We Teach This

This won't have much impact as it extends the current behaviour by adding
functionality.

The only breaking change is that `resolutions` is being considered all the time,
but that won't surprise people, this will make yarn behaviour simply more
consistent than before (see the comment on `--flat` above).

The term "resolution" has the same meaning as before, but it is not under the
sole control of yarn itself anymore, but also under the control of the user
now.

This is an advanced use of yarn, so new users don't really have to know about
it in the beginning. Still, it is meant to be used on a potential regular
basis, in particular when some packages a project depends on have problems
in their how dependencies.

Thus it would make sense to have a bit of the documentation talking about
this use case and underlying the fact that *resolutions* are mostly here
on a temporary basis.

# Drawbacks

## Teaching

It makes yarn behaviour a bit more complex, even though more useful. So it
can be difficult for users to wrap their head around it. The RFC submitter has
seen it happen many times with maven, which is quite complex but complete in
its dependency management. Users would get confused and it can take time to
understand the implications of manipulation the `resolutions` field .

# Alternatives

## Global nested dependencies resolution

Starting from an example, this solution would take the following form in the
`package.json` file:
```json
  "devDependencies": {
    "@angular/cli": "1.0.3",
    "typescript": "2.3.2"
  },
  "resolutions": {
    "typescript": "2.0.2"
  }
```

yarn would use `typescript@2.0.2` for the whole project and that's all.
The same kind of consideration (outside of the glob pattern thing) should be
followed that in the RFC selected solution.

## Mapping version specifications

This is a kind of simplified solution to the "out-of-scope scenario" presented
in the motivations section above (it maps versions but not dependency names).

It was proposed in this
[comment](https://github.com/yarnpkg/yarn/issues/2763#issuecomment-301896274).

It is similar to the previous alternative but with a version specification
in the package designation. This would take this form in the `package.json`:
```json
"devDependencies": {
    "@angular/cli": "1.0.3",
    "typescript": "2.2.2",
    "more dependencies..."
  },
  "resolutions": {
    "typescript@>=2.0.0 <2.3.0": "typescript@2.3.2"
  }
```

yarn would then replace matching version specifications with the user's one.
For example a dependency normally resolved to `typescript@2.2.2` would be
resolved in practice to `typescript@2.3.2`.

## Mapping version specifications as well as packages name

Same as the tow above but with a different name on the right-hand side of the
*resolution*:
```json
"devDependencies": {
    "@angular/cli": "1.0.3",
    "typescript": "2.2.2"
  },
  "resolutions": {
    "typescript@>=2.0.0 <2.3.0": "my-typescript-fork@2.3.2"
  }
```

or even:
```json
"devDependencies": {
    "@angular/cli": "1.0.3",
    "typescript": "2.2.2"
  },
  "resolutions": {
    "typescript": "my-typescript-fork",
  }
```

and the version specification would be conserved.

# Unresolved questions

## Package designation and single `*`

I am not totally convinced that using a single `*` in a package designation
is a good idea. Mostly because it introduces much for uncertainty to what it is
going to be applied. In particular, I'm thinking about defining a *resolution*
like that and latter adding a dependency which has a dependency that matches
but is not intended to. I feel like it adds a lot of mental weight for the
user to manipulate single `*`.

## `--flat` option

When used with the `install` command, the behaviour of `--flat` with respect
to this RFC is clear: it will populate the `resolutions` for all packages with
`**/dependency` as package designations.

But when used with the `add` command, should it apply only to the nested
dependencies of the added dependency? I think that it would make sense.

## `check` command

The `check` command will need to be modified to take resolutions into account.
This must be specified more precisely and will be added to the RFC in the
near future.

## Extensions of the proposition

Actually, the two alternatives "Mapping version specifications" and
"Mapping version specifications as well as packages name" could be adapted
to the current proposition to support these uses cases as well.
Not sure it is a good idea but it could be useful maybe...