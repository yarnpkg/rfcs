- Start Date: 2017-05-04
- RFC PR:

# Summary

This document specify a new command, that can be used to execute subcommands inside a project workspaces.

# Motivation

With the addition of the Workspace feature, it will become handy to be able to execute commands inside other workspaces than the one in the current directory.

# Detailed design

This RFC suggests to add the following commands:

## `yarn exec <binary-name> ...`

Execute a shell command inside the same environment than the one used when running scripts. For example, running `yarn exec env` will print something similar to this:

```
PWD=/path/to/project
npm_config_user_agent=yarn/0.23.4 npm/? node/v7.10.0 darwin x64
npm_node_execpath=/usr/bin/node
...
```

## `yarn workspace <workspace-name> <command-name> ...`

This command will execute the specified sub-command inside the workspace that is being referenced by `<workspace-name>`.

Recursion aside, it's essentially an alias for:

```
$> (cd $(yarn workspace <workspace-name> exec pwd) && yarn <command-name> ...)
```

# Drawbacks

- We will still have the issue of requiring the `--` separator to forward any command line option (`yarn workspace test install -- --production`). It's an important issue, something we really should tackle sooner than later.
