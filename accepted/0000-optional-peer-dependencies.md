- Start Date: 30 Oct 18
- RFC PR: https://github.com/yarnpkg/rfcs/pull/105
- Yarn Issue: n/a
- Champion: Maël Nison (@arcanis - [Twitter](https://twitter.com/arcanis))

# Optional Peer Dependencies

## 1. Which Problem Does It Solve

> I'm a library author. I want my package to provide an optional integration with some other library, but I don't want to have to ship it.

The current solution to the problem described above is peer dependencies. A package will list the optional dependency as peer dependency, meaning that it's to the package consumer to also provide the library that should be used.

While it currently already works, it causes semantical issues: are peer dependencies meant to be optional or not? Should a missing peer dependency cause a warning? An error? Nothing at all, and expect the runtime to check that they are available?

## 2. Detailed Design

Yarn will introduce a new field, `peerDependenciesMeta`. This field will be a dictionary that will allow adding metadata to the peer dependencies. As a first step, we'll only add the `optional` metadata field.

```json
{
  "peerDependencies": {
    "lodash": "*"
  },
  "peerDependenciesMeta": {
    "lodash": {
      "optional": true
    }
  }
}
```

## 3. How We Teach This

- Being stricly backward compatible it doesn't require us to push changes onto our users, so the teaching should be simple and spread on the long time.

## 4. Drawbacks

- It requires adoption from other package managers, otherwise there's a risk to fracture the ecosystem if they decide to go with (for example) naming the field `peerDependenciesSettings`.

## 5. Alternatives

- We could introduce an `optionalPeerDependencies` key.

  - It would be one more key to teach users.

  - This wouldn't be backward compatible - most package managers would have no idea what to do with such a key and would ignore it entirely, breaking the tree because of the hoisting.

  - Semantic isn't clear if a dependency is both in `peerDependencies` and `optionalPeerDependencies`.

  - It would have a completely different meaning from the already existing `optionalDependencies` field.

- We could add an `optional:` protocol

  - It's been noted it could potentially cause issues on older package managers.

- We could overhaul how the dependencies are defined, and fix them once and for all.

  - This sounds a huge undertaking for a problem relatively minor at the moment.

  - It seems unlikely we can reach a consensus in a reasonable timeframe.
