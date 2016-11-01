- Start Date: 2016-10-31
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

> The ability to dynamically determine packages based on variable substitution.

While there could be many great use cases for variable substitution, I will
address the two most important variables that I've encountered, `platform` and
`cpu`. The syntax for substitution within dependencies is `.(variableName)`,
which I'll explain later.


### What variable substitution does:

``` json
"devDependencies": {
   "rebel": "ourGitUrl#0.4.4.(anyOs).(anyCpu)"
}
```

Would result in `yarn` interpreting the `package.json` as if it had been written as:

``` json
"devDependencies": {
   "rebel": "ourGitUrl#0.4.4.darwin.64"
}
```

if you are on Mac OS, 64 bit. 

I'm seeking feedback on the *exact* parameters/variables names to use (it has
been suggested that we simply copy what Brew does).


# Motivation


This enabled blazing fast installs for what would normally take as much as `30`
minutes on `npm` - for `postinstall`-heavy packages.


# Detailed design

This convention is not only great for package installers (users), but great for
package authors because they can configure travis builds to automatically
upload successful builds to `ourGitUrl#0.4.4-.(anyOs).(anyCpu)` where
`platform`/`cpu` matches the platform/architecture that travis built on.  So,
that `package.json` snippet above is actually something that package authors
could commit and publish, knowing that it will work everywhere when people
begin to depend on their packages. There is no way to do this with `yarn` or
`npm` today that I know of (not even with the `os`/`cpu` fields).

You can imagine allowing this usage of `yarn`, in order to include prebuilts
for the exact architecture you want:

```
yarn install --vars -platform=linux -cpu=64
```

But what about OSs/CPUs that we haven't built for? What about people who use
the `npm` client to install a package with `yarn` variables in it? The proposed
variable syntax is intentionally chosen to be `.(varName)`, because `.(foo)` is
valid syntax for git tags, even if it's not substituted with anything.

So the solution, is that in addition to having travis publish
`#0.4.4-.darwin.64` that contains the prebuilt version of the package for
`darwin`, we _also_ publish the _not_-prebuilt version of the package from
travis with `#0.4.4-.(anyOs).(anyCpu)` - yes, the literal string
`"#0.4.4-.(anyOs).(anyCpu)"` including parens. That means `npm` clients can
_still_ depend on our packages that contain `yarn` specific substitution
features, it's just that they end up rebuilding the package from source. It's a
nice incentive to use `yarn`, without breaking compatibility with `npm`, and
hopefully `npm` can adopt the convention if people find it useful.

We could also allow installation of *some* packages to avoid variable
substitution, and therefore force rebuilding of those packages.

Feature 1: Ability to control how one *`package.json`*'s variables are substituted:

In this example, every package will have `linux` and `64` substituted for
`platform`/`cpu` respective, *except* `rebel`'s `package.json`.

```
yarn install --vars -platform=linux -cpu=64 --vars-for-package rebel --platform=anyOs -cpu=anyCpu
```

Feature 2: Ability to control how one *dependency* has its variables
substituted, regardless of which `package.json` depends on it.

```
yarn install --vars -platform=linux -cpu=64 --vars-for-dependency rebel --platform=anyOs -cpu=anyCpu
```


# How We Teach This

This is a pretty simple concept to explain - it's basically template substitution.

# Drawbacks

`yarn` git tag version constraints should also work here, although these would
not be compatible with `npm` (sadly), but the `npm` incompatibility is only due
to `npm` not supporting git tag ranges, and that has nothing to do with
variable substitution.

```
"dependencies": {
   "rebel":
      "gitUrl#0.4.(os).(cpu) - gitUrl#0.5.(os).(cpu)"
}
```

The main drawback is that while it empowers developers to depend on
*potentially* pre-built packages and while `travis` enables publishing them in
the proper form very easily, `package.json` files become strange looking, and
`package.json` files start to leak some of the implementation details about how
a package should be installed.

# Alternatives

The impact of not solving this problem *somehow* is that `yarn` and `npm` won't
be capable of modeling languages outside of JS, or JS projects that integrate
with other languages which have heavey `postinstall` steps.

Instead of saying: "I depend on `rebel`, and here's where you can get the
prebuilt binary", it might be better to say, "I depend on rebel", and then
allow `rebel` itself to specify where to get the prebuilt binaries. Or perhaps,
there should be a way for any of those entities to configure where that binary
is and then it should be negotiated. But I've made the proposal as is,
specifically because it may be incredibly easy to implement, and even works
with `npm` in most cases!

# Unresolved questions

- This proposal has `npm` compatibility via full rebuilds and with `yarn` we
  can substitute `.(os)`/`.(cpu)` across all packages, or no packages at all,
  but how do we substitute variables for only the subset of packages that
  _have_ a tag for that os/cpu? This could be solved much later, but I imagine
  that the `yarn` client could _first_ try substituting `os/cpu` and then if no
  git tag is found, it could fall back to the literal string
  `#4.0.0.(anyOs).(anyCpu)`, and then finally `#4.0.0`.


