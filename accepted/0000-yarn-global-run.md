- Start Date: 2017-09-12
- RFC PR: (leave this empty)
- Yarn Issue: [yarnpkg/yarn#4392](https://github.com/yarnpkg/yarn/issues/4392)

# Summary

Adding a `yarn global run` command, which is a more generic version of 
`yarn create`. It will globally install the package and run its binary with the
arguments that are passed after `yarn global run`.

# Motivation

Some packages that are similar in design to `create-react-app` ([vue-cli](https://yarn.pm/vue-cli),
[preact-cli](https://yarn.pm/preact-cli), [@angular/cli](https://yarn.pm/@angular/cli)
...) don't have a name that's prefixed with `create-`, and thus prevent the usage of
`yarn create`, even though it would be perfectly suitable for it. 

# Detailed design

The design would be exactly the same as `yarn create`, but without the prefixing of
`create-`. 

# How We Teach This

Yarn run and Yarn global already exist, so this is not introducing new concepts, 
especially since `yarn create` already exists.

A blogpost with the reasoning behind this change

# Drawbacks

It can cause confusion with regard to `yarn create`, since it has a very similar
goal. 

# Alternatives

Npx already exists and fills this position for now. 

Adding a flag like `--exact` to `yarn create` so that the `create-` prefix isn't 
automatically added

A separate binary `yr` with this behaviour

Yarn create can use a package without `create-` prefix if it doesn't exist

> This could be having a "confirm" prompt

# Unresolved questions

- will the installed package be installed globally
- is the name good
- how does this conflict with `yarn create`