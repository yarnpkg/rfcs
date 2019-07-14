- Start Date: 2018-10-21
- RFC PR:
- Yarn Issue:

# Summary

I want to add the option to add ddocumentation for packages and their chosen version ranges.

# Motivation

My team and I found ourselves with the need to document why some packages got
pinned down to specific versions.

## Examples where a feature like this is missing

1. We took a decision to stay with React 15 and not to migrate to React 16 and
this lead to many packages staying on older versions as well.

There's currently no convenient way to signal the next developer who would like to
upgrade the versions of packages in the project, that they should keep
some on older versions and why they should do it.

2. Sometimes a bug is found even in minor version updates of packages and there is
no convenient way to signal developers why a package was pinned to a certain minor version.

for example:
```
"some-package": "1.0.4"
```
Where `1.0.6` is available.

A developer that sees this kind of version pin usually starts investigating the
git history and people who are involved in the pinning of the package.

3. We had a few cases, where we cloned a repository and fixed or modified an npm library.

For the time between our new code is ready and the time it's merged(sometimes it takes weeks or months)
We used our forked git repository as the version of the package:
```
"some-package": "github:user/repo"
```
There's currently no convenient way to document when should the git repository URL be replaced
with the newest version of a package like this (probably when the corresponding PR is merged).

4. Some packages appear in package.json to be installed in node_modules for the sake
of being used by other libraries and they are not used in the source code of the
project itself.

It's very tempting to delete such packages because they are "not used anywhere" but a
convenient way to document this could be very benefitial to the user.

5. Some packages has breaking changes in their new versions that are not easy to
cope with. In that case, it would be useful to signal a developer there's a ticket to upgrade
that specific package and what work has been done on it.

# Detailed design

My idea is to save the documentation of packages in an additional optional per package `package.json` entry
with the following data structure:
```
"dependencies-documentation": {
  "some-package": "this package is used by some-package-b",
  "some-package-b@1.1.3": "pinned because of the following bug: http://www.github.com/package/issues/20"
}
```
Upon running commands like `yarn upgrade` or `yarn upgrade-interactive`, comments
would appear to the user as info and warnings and confirmations about pinned versions
would run.

Upon running commands like `yarn remove` the package's comment would appear and if the
the user still wants to delete it, it would also be deleted from `dependencies-documentation`.

If a package gets deleted manually and `yarn` is run where it is still in `dependencies-documentation`
the comment should be shown to the user with a prompt to delete it or add it back.

Warnings should aslo run if there is a version mismatch between the package's version in `package.json`
and `dependencies-documentation`.

All these features would only run if `dependencies-documentation` is added manually by the user.

If the user doesnt add this field manually, nothing should change in how `yarn` works.
 
# How We Teach This

I believe adding it to the documentation and adding a message to
`yarn upgrade` and `yarn upgrade-interactive` that would suggest the user use it with a link to the
documentation is enough.

# Drawbacks

Having versions appear in two places can be problematic both for users and for certain
commands.

# Alternatives

* I thought to use a simple "packages.md" file or adding it to the readme, but then it
gets lost and it's too easy to miss.

* Adding comments to package.json could work but it's not possible because too many tools
expect this file to be a simple JSON and not not a JSON5 file.

* Another option would be to use repository commits messages to keep track of commits where package.json got
modified, but this can also be missed and not easy to maintain.

* Simply adding this field to `package.json` without any enforcing from `yarn` might be enough
but 

* We can use an extra library to do this work. But it feels importent enough to be added to `yarn` as a native feature.
