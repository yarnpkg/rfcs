- Start Date: 2017-03-13
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Rename `yarn clean` command.

# Motivation

1. People might expect `yarn clean` to do a different thing (to clean build artifacts) 
than it actually does 
(cleans node modules of some files and sets the system to clean it automatically in the future). 
This is a very natural expectation because it is usually possible to do, e.g., 
`yarn watch`, `yarn dist`, and of course, `yarn clean` seems like a logical command 
(compare with gradle clean, mvn clean, gulp clean).

 Next, when `yarn clean` is executed a user realizes that it is not the correct command 
but it does not seem to do anything drastic (probably just cleans some yarn caches) and 
forgets about it. 
(Of course, the right thing to do would be to look at https://yarnpkg.com/en/docs/cli/clean instead.) 
The result often is that the project is broken and the bug is hard to detect. 
There are a lot of such bugs in this project and more elsewhere (example: twbs/bootstrap-sass#1097).

 The name is confusing and the best way to stop confusion seems to be renaming the command.

2. Also, it is nice to be able to run a user-defined `yarn clean` if you already can run a user-defined `yarn build`.

# Detailed design
The command is to be renamed to `pruneModules`.

Next, `clean` should be available for redefinition by user scripts.

It should be more or less safe after that to produce an error in a standard way when `clean`
is executed but the user script is missing. This is because the command is not usually run
during a build but is mostly executed manually and its state persisted in cvs as a `.yarnclean` file.


# How We Teach This

The possible new name candidates are, e.g.,:
`strip`, `shrink`, `stripModules`, `shrinkModules`, `cleanModules`,
`stripPackages`, and even `yarn-clean`.

The documentation should only reflect the change of the command name.

Release notes should provide a note about a possible breaking change.

# Drawbacks

We should not do this if yarn does not want to promote using `yarn command`
for user defined scripts. Note that then the existing usage of such commands 
should also be deprecated.

# Alternatives
1. Produce big and nice warnings when the command is used 
and on subsequent installation of modules (listing cleaned/ignored files).
(This does not solve point 2. of the Motivation.)
2. Deprecate `yarn command` for user-defined scripts. (So that only `yarn run command` is supported.)
3. Deprecate running user-defined commands through yarn altogether (and optionally provide
a different default command for running user-scripts, e.g., `yarun`).
4. Some alternative proposed new name candidates are:
`delete-module-bloat`, `autoclean`, `delete-package-assets`, `remove-module-files`,
`enable-advanced-auto-disk-space-optimizations`, `prune`,
`strip`, `shrink`, `stripModules`, `shrinkModules`, `cleanModules`,
`stripPackages`, `yarn-clean`.
