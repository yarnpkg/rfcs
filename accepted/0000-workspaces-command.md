- Start Date: 2017-05-04
- RFC PR:

# Summary

This document specify a new command, that can be used to execute subcommands inside a workspace projects.

# Motivation

With the addition of workspaces, it will become handy to be able to execute commands inside other projects than the one in the current directory.

# Detailed design

This RFC suggests to add the two following commands:

```
$> yarn project <project-name> pwd
$> yarn project <project-name> <command-name> ...
```

The first one will print the project directory for the package referenced by `project-name`.

The second one will be an alias for:

```
$> (cd $(yarn project <project-name> pwd) && yarn <command-name> ...)
```

# Drawbacks

- We will still have the issue of requiring the `--` separator to forward any command line option (`yarn project test install -- --production`). It's an important issue, something we really should tackle sooner than later.

# Alternatives

- Is `yarn project` a good name? I considered `yarn workspace`, but we're not operating on the workspace itself, just its projects. I feel like a `yarn workspace` command should only affect the root package.json (the one who contains the `workspaces` directive).
