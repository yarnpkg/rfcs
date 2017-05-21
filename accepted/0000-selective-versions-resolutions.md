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

## Variations of the original problem

Similarly, even using such a content for `package.json`:
```json
 "devDependencies": {
    "@angular/cli": "1.0.3"
  }
```

The need could arise for forcing the use of `typescript@2.3.2` (or
`typescript@2.1.0` for that matter).

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
file to be considered all the time and on a per-package basis.

When a nested dependency is being resolved by yarn, if the `resolutions` field
contains a version for this package, then this version is used instead.

All the examples are given with exact dependencies, but note that putting a
non-exact specification in the `resolutions` field should be accepted and
resolved by yarn like it usually does.

## Example

For example with:
```json
 "devDependencies": {
    "@angular/cli": "1.0.3"
  },
  "resolutions": {
    "typescript": "2.3.2"
  }
```

yarn will use `typescript@2.3.2` for every nested dependency to `typescript`
and will behave as expected with respect to the `node_modules` folder by not
duplicating typescript installation.

## Relation to non-nest dependencies

The `devDependencies` and `dependencies` fields always take precedence over the
`resolutions` field: if the user defines explicitly a dependency there,
it means that he wants that version, even if it's specified with a non-exact
specification. So the `resolutions` field only applies to nested-dependencies.

## Relation to the `--flat` option

The `--flat` option becomes thus a way to populate the resolutions field for
the whole project, as it already does.
But the `resolutions` field is always considered by yarn, even if `--flat` is
not specified.

Incidently, this resolves this strange situation when two developers would be
working on the same project, and one is using `--flat` while the other is not,
and they would get different `node_modules` contents because of that.

## `yarn.lock`

This design implies that it is possible to have for a given version
specification (e.g., `>=2.0.0 <2.3.0`) a resolved version that is incompatible
with it (e.g., `2.3.2`).
It is acceptable as long as it is explicitly asked by the user.

It is currently the case that such situation would make yarn unhappy and
provoke the modification of the `yarn.lock` (see 
[yarnpkg/yarn#3420](https://github.com/yarnpkg/yarn/issues/3420)).

This feature would remove the need for this behaviour of yarn.

## Warnings in logs

yarn would need to warn about the following situations:
- Unused resolutions
- Incompatible resolutions: see the above section about `yarn.lock`.
Incompatible resolutions should be accepted but warned about since it could
lead to unwanted behaviour.
- ? (see open questions below)

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
it in the beginning.

# Drawbacks

## Teaching

It makes yarn behaviour a bit more complex, even though more useful. So it
can be difficult for users to wrap their head around it. The RFC submitter has
seen it happen many times with maven, which is quite complex but complete in
its dependency management. Users would get confused and it can take time to
understand the implications of manipulation the `resolutions` field (even
though, the chosen solution, compared to the alternatives below, is much
simpler).

## Package management paradigm

Yarn and npm users are highly used to the idea that a dependency can be
present many times in the `node_module`, depending on which package needs it.
This has advantages and disadvantages, but it is one of the specificity of the
npm ecosystem package management.

In this light, taking such as design decision puts yarn a bit farther to such
way of doing thing, and it could be considered a bad direction to go toward.

Some of the alternatives below actually take this into consideration, but are
a bit more complex in terms of expressiveness, so were not chosen by the RFC
submitter (see open questions below too).

# Alternatives

There is at least one alternative to the proposed solution, more complex but
more expressive.

## Nested dependencies resolution per dependency

Starting from an example, this solution would take the following form in the
`package.json` file:
```json
  "devDependencies": {
    "@angular/cli": "1.0.3",
    "typescript": "2.3.2"
  },
  "resolutions": {
    "@angular/cli": {
      "typescript": "2.0.2"
    }
  }
```

yarn would use `typescript@2.0.2` only for `@angular/cli` (so in
`node_modules/@angular/cli/node_modules`), but keep `typescript@2.3.2` in
`node_modules/typescript`.

Basically, this enables the user to specify versions for nested dependencies,
but only in the context of a given dependency.

The fields of the `resolutions` field must only refer to existing entries in
`devDependencies` and `dependencies`.

Of course, if the same version of a nested dependency is used for many
dependencies, yarn will behave as always by keeping it directly in
`node_modules`.

## Mapping version specifications

This is a kind of simplified solution to the "out-of-scope scenario" in the
motivations section above (it maps versions but not dependency names).

It was proposed in this
[comment](https://github.com/yarnpkg/yarn/issues/2763#issuecomment-301896274).

Everything is not totally clear to me, but the idea would be to map a given
version specification to another one.
This would take this form in the `package.json`:
```json
"devDependencies": {
    "@angular/cli": "1.0.3",
    "typescript": "2.2.2",
    "more dependencies..."
  },
  "mappings": {
    "typescript@>=2.0.0 <2.3.0": "typescript@2.3.2"
  }
```

yarn would then replace matching version specifications with the user's one.
What is problematic with this is that the user has to know that `@angular/cli`
is exactly expressing its dependency to `typescript` as `>=2.0.0 <2.3.0`.

This makes such mappings hard to maintain because they can become ignored if
`@angular/cli` is upgraded and its dependency specification changes, while
the other solutions would only result in 

# Unresolved questions

## Is this expressive enough?

As explained in the alternative solutions section, it would be much more
expressive and coherent with the npm ecosystem package management paradigm
to use nested dependency resolutions per project dependency.
Would the loss of simplicity acceptable maybe?

## Warnings in logs

Should yarn warn the user about an incoherence between an explicit dependency
and a resolution. For example if the user specify a dependency to
`typescript@2.3.2` and the resolutions field contains `typescript@2.3.0`.
For sure if the above alternative solution is chosen, this wouldn't make sense.

Should we warn if a resolutions is incompatible, but still upper-bounded?
For example, forcing version `a@2.3` while a dependency needs version `a@2.2`
is usually less problematic than forcing version `a@2.2` while a dependency
needs version `a@2.3`.
The problem with differentiating these situations is that yarn to start giving
lots of semantics to versions and it can give false certainty to the user than
a problematic situation is not problematic. So it may be better to always warn
about incompatible resolutions.
