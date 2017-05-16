- Start Date: 2017-05-04
- RFC PR:

# Summary

This document specify a new command, that can be used to execute subcommands inside a project workspaces.

# Motivation

With the addition of the Workspace feature, it will become handy to be able to execute commands inside other workspaces than the one in the current directory.

# Detailed design

This RFC suggests to add the two following commands:

```
$> yarn workspace <project-name> pwd
$> yarn workspace <project-name> <command-name> ...
```

The first one will print the project directory for the package referenced by `project-name`.

The second one will be an alias for:

```
$> (cd $(yarn workspace <project-name> pwd) && yarn <command-name> ...)
```

# Drawbacks

- We will still have the issue of requiring the `--` separator to forward any command line option (`yarn workspace test install -- --production`). It's an important issue, something we really should tackle sooner than later.
