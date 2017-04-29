- Start Date: 2017-04-29
- RFC PR: 
- Yarn Issue: 

# Summary

Add a `project:` dependency option, which will resolve tarball packages against
the project root instead of the current directory. Unlike `file:`, the tarball
will not be cached.

# Motivation

Currently, when you want to add a dependency to a 'local' package you have to
use a `file:` dependency. To have some form of a reusable dev setup you will
likely use a relative path. Problem is that this might not resolve correctly
on transitive dependencies:

```
/shared/foo
  
/shared/bar
  dependencies: foo "file:../foo/build/foo.tgz"

/module/qux
  dependencies: bar "file:../shared/bar/build/bar.tgz"
```

In this case package `qux` will look for `foo` at `/module/foo/build/foo.tgz`. 
This particular layout could use `../../shared/foo/build/foo.tgz` instead but
this will still break if not all packages are at the same level in the directory
tree.

The proposed solution is to use instead

```
/shared/foo
  
/shared/bar
  dependencies: foo "project:shared/foo/build/foo.tgz"

/module/qux
  dependencies: bar "project:shared/bar/build/bar.tgz"
```

Now transitive dependencies can always be resolved correctly. A file like
`.yarn.project` could mark the project root against which paths will be 
resolved.

# Detailed design

An initial implementation is available.

A 'project' is a directory containing multiple packages somewhere below it. The
root is marked by the existence of a file `.yarn.project`. Something similar is 
used by modern build systems:

- [Bazel](bazel.build): 'Workspace' contains 'packages'. Labels are of the form
  '//module/qux:build_name'. Root is marked with a `WORKSPACE` file.
- [Buck](buck.build): 'Project' contains 'build rules'. Build targets are of the
  form '//module/qux:rule_name'. Root is marked with a `.buckconfig` file.
- [Gradle](gradle.org): 'Root project' contains 'sub projects'. 
  Tasks use ':module:qux:taskName'. Root is marked with a `settings.gradle`
  file.

I don't know how they handle accidentally nested projects. This might be worth
 looking into.

Unlike any other resolvers, the `project:` resolver should check the hash of the
file during integrity checking. If the hash does not match the resolved version
in the lockfile Yarn should not bail out. This is so we can be sure we are 
always using the current version of the local tarball. This should throw an 
error if you run with `--frozen-lockfile`.

Resolving could use the already existing `LocalTarballFetcher`.

# How We Teach This

I think the easiest way to explain this functionality would be as a comparison
to the `file:` type.

This functionality is completely opt-in, so I foresee no large problems with
use acceptance, given that documentation is available.

# Drawbacks

Basically ignoring version numbers is somewhat orthogonal to the design of NPM.
I think this is exactly what you want when using a monorepo though and think it
is worth it.

This will obviously not work on published packages.

Supporting only tarballs still requires the user to build the actual tarball
from the current sources. You will still need some external application to
handle building in the correct order otherwise you might still use an outdated
version.

# Alternatives

There are a couple of alternatives:

##### Symlinks
These can be created using `yarn link`, the proposed `link:` syntax in
[#34](https://github.com/yarnpkg/rfcs/pull/34) or in newer NPM versions using
`file:`.
 
The problem with all of these is that `require` will resolve against the real
file location, likely giving completely different results than using an actual
package. This basically splits dependency resolving into multiple completely 
independent parts. In most cases this will be unwanted.

##### Using `file:` with tarball
The current `file:` implementation is incredibly broken. It could (should) be
fixed to not cache tarballs using the filename only. This will still not work
for transient dependencies. If you don't have transient local dependencies this
should work fine though.

##### Using `file:` with a directory
Will also not work with transient dependencies, but now you also have the 
problem of detecting changes. If you rely on the version in `package.json` you
might get unexpected results if you make changes and forget to update the 
version number (new installs will use a different version than updated installs)

##### Linking to a package instead of a tarball
This would be nice, but requires Yarn know how to build a package. There is 
already `pack` for packaging, but there is some kind of `prepack` script hook
missing to actually run some build steps. It would be a lot more work to get
working. Detecting if a package needs to be rebuild will be challenging.

##### Transparently use a local version if found in the project

If you declare any dependency on `foo` it will automatically use the local
version if there is any package in the project called `foo`. I guess this will
be pretty unexpected behaviour. You would need to know all local packages in
advance and how to build them if necessary.

# Unresolved questions

The current implementation stores the tarball hash in the lockfile. This makes
sure that everybody is using the same version at all times (if they differ you
will get a different lockfile or a build error if using `--frozen-lockfile`).
It also requires that your build system produces bit-for-bit equal artefacts in
every environment.

This makes it a little harder to get working correctly, but it also opens some
of the advanced caching functionality exposed by Bazel/Buck/Gradle (they
retrieve artefacts from the build server if the hash matches, skipping the need
for a local build).

It would be possible to leave the hash out of the lockfile but this means you 
don't get a warning if somebody is producing different tarballs for some reason.
