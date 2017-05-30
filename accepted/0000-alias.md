- Start Date: 2017-05-14
- RFC PR: (leave this empty)
- Yarn Issue: https://github.com/yarnpkg/yarn/issues/1628

# Summary

Add an alias system similar to `git alias` to the yarn project.

# Motivation

Everyone has different needs, It is right to don't have multiple commands that does the same thing, but at the same time I think it is right to give the possibility to the user to config their aliases. One example is that I prefer type `git dc`
instead of `git diff --cached`, it is an habit, I know that and I will never ask the git project to support the `dc` alias.

# Detailed design

We can add a `yarn alias` command and we can save every alias in the `.yarnrc` file.
After this first step, we can:
- remove every unsopprted alias that now prints 'Did you mean <something>?' because it is very confusing and weird to support them.
- bring some consintency between commands (e.g. use list or remove everywhere, use long names everywhere, etc..)

# How We Teach This

I think we should just say that it is like `git alias` and for skilled users suggest to save the yarnrc file in their dotfiles.

# Drawbacks

Maybe everyone can do on the machine using the alias system of every os.

# Alternatives

We can create a meta file in the project, we can commit in the project but we can't save it in our dotfile project

# Unresolved questions

Like git does, we should translate through the alias system before *everything* and support:
- subcommands
- commands with flags
I still don't know how hard can be this with the yarn codebase.
