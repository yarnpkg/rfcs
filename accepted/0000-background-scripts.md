- Start Date: 2018-10-10
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Let's give yarn the ability to watch over processes running in the background.

Currently, `package.json` has a `scripts` field that allows devs to execute processes in the foreground. This is useful for build scripts, running tests, starting servers etc:

```
"scripts": {
    "start": "node lib/index.js",
    "test": "jest",
    "build": "babel src --out-dir lib"
}
```

Let's add a new `backgroundScripts` field to `package.json` where we can define long lived processes that yarn will run in the background:

```
"scripts": {
    "test": "jest",
    "build": "babel src --out-dir lib"
},
"backgroundScripts": {
    "express-server": "nodemon lib/index.js",
    "babel-compiler": "yarn build --watch",
    "mock-json-server": "json-server --watch db.json"
}
```

Let's also provide a CLI API to view and control the status of the processes:

![example status API](https://i.imgur.com/2a1Mhx1.png)
_Example output for `yarn bg status`_

# Motivation

A typical workflow is to define scripts in `package.json` to orchestrate the local dev environment. The scripts are typically run in the foreground (e.g. `yarn start`).

As applications grow more complex, more _stuff_ is needed to build and run the application (e.g. build scripts, mock servers etc). Having scripts watch code and execute this logic in the background allows for quicker iteration cycles.

Today, this is achieved by either:

- running multiple scripts in different tabs/tmux windows
- manually sending to the background (e.g. `yarn build --watch &`)
- specialized tooling that combine both the building and the running of code. Examples include:
  - https://github.com/jaredpalmer/backpack
  - https://github.com/kmagiera/babel-watch

The pattern of combining the build script and the application run script (a la create-react-app) makes for a simple developer experience. However, extending beyond doing two things in the `yarn start` script is harder, mixes concerns and requires developers to write custom logic in startup scripts.

Let's generalize this approach by allowing arbitrary scripts to run independently of each other in the background, managed by yarn.

## Why should yarn include this?

This could be achieved by a new package independently of yarn. Here's some reasons why yarn should include this:

1. There is prior art for yarn providing first class support for features that could be achieved outside of yarn (see: workspaces/lerna). This allows for richer API integration, leading to new usages and performance improvements.
2. yarn already runs arbitrary scripts - this new feature could be thought of as an extension of the existing API.
3. Process management is hard to get right, especially cross platform. By having yarn manage this, and by leveraging the [existing logic](https://github.com/yarnpkg/yarn/blob/09ceb0e594b694bb773faeb23fd69174f6ac9623/src/util/child.js#L61) around running scripts, we can provide a strong guarantee to developers that their scripts will work as expected, be system agnostic, thus providing more overall ecosystem stability.

_Editor's Note:_ At this stage, I'm unsure what the API would look like for nested packages in workspaces (maybe this is just disallowed?), but by having the context of where a background script is running in relation to the rest of the repo, and how could be helpful. In particular, maybe yarn auto-starts background scripts when a package newly linked? This is some of the thinking behind point (1) above.

## Why shouldn't yarn include this?

1. Process management is hard :P
2. This is a non trivial addition of functionality and source of new bugs for maintainers. The surface area of the yarn API will be expanded - a decision that should not be made lightly.
3. A similar API could be achieved without needing to include this in yarn.

Point (3) is significant, and should be highly weighted.

# Prior Art

Here's some prior art that this RFC takes inspiration from:

- https://pgctl.readthedocs.io/en/latest/
- http://pm2.keymetrics.io/docs/usage/quick-start/
- http://supervisord.org/
- https://circus.readthedocs.io/en/latest/

# Detailed design

> _TODO:_ At this stage, this just a sketch of the API without discussion of how it will be implemented internally.

We must be able to do the following:

- Start/Stop/Restart individual or all background scripts
- View the output/logs of background scripts

Nice to have:

- A flag to run any of the background scripts in the foreground (so we can attach a tty for debugging)

## API Design

We will expose a new subcommand, `yarn pg` to allow developers to interact with their background scripts.

### Starting/Stoping/Restarting background scripts

| Command             | Description                                                            |
| ------------------- | ---------------------------------------------------------------------- |
| `yarn bg start`     | starts all scripts defined in `package.backgroundScripts`              |
| `yarn bg start foo` | starts only the `foo` script as defined in `package.backgroundScripts` |

There must be equivalent `stop` and `restart` versions of the above commands

### Monitoring script up/down status

| Command                 | Description                                                                        |
| ----------------------- | ---------------------------------------------------------------------------------- |
| `yarn bg status`        | Prints the status of all background scripts defined in `package.backgroundScripts` |
| `yarn bg status foo`    | Prints the status of the `foo` background script                                   |
| `yarn bg status --json` | Prints the status of all background scripts in json format                         |

### Viewing logs

| Command                   | Description                                                                                  |
| ------------------------- | -------------------------------------------------------------------------------------------- |
| `yarn bg logs`            | Prints the combined output for all background scripts defined in `package.backgroundScripts` |
| `yarn bg logs foo`        | Prints the output for the `foo` background script                                            |
| `yarn bg logs foo --tail` | Follows the output for the `foo` background script                                           |

## How/where state is stored

As part of process management, we will need to store the following on disk:

- a file containing the pid for each background script started by yarn
- log(s) of the background script stdout/stderr

There are a few options:

1. Storing in a `.yarnbg` directory, relative to the current working directory of the repo

   Notes:

   - Would require developers to add a new line `.gitignore` (prior art: node_modules)

2. Storing in a `node_modules/.yarn-background-scripts` directory, relative to the current working directory of the repo

   Notes:

   - Avoids dirtying the repo, as seen in (1)
   - There's prior art for storing extra things in `node_modules` (see: `.yarn-integrity`)
   - Risky since `rm -rf`'ing `node_modules` is commonly done - which would wipe out the state.

3. Storing in `~/.config/yarn/background/<REPO_DIR>`

   Notes:

   - Avoids dirtying the repo, as seen in (1)
   - Incurs a layer of indirection to where the state lives
   - Yarn already stores link state in `~/.config/yarn/`

4)  Storing in `~/.local/share/yarn/background/<REPO_DIR>`

    Notes:

    - Basically the same as above, but more semantically suited to short lived data files

**Unresolved Question:** `<REPO_DIR>` is actually dangerous - I can imagine an edge case where a dev renames a directory, such that the pointer to the state is now lost. Option (1) above seems safest, but if dirtying the repo is unacceptable, a generated persisted ID could work in place of <REPO_DIR>, at the cost of a more complex implementation.

# How We Teach This

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing npm patterns, existing Yarn
patterns, or as a wholly new one?

Would the acceptance of this proposal mean the Yarn documentation must be
re-organized or altered? Does it change how Yarn is taught to new users
at any level?

How should this feature be introduced and taught to existing Yarn users?

# Drawbacks

Why should we _not_ do this? Please consider the impact on teaching people to
use Yarn, on the integration of this feature with other existing and planned
features, on the impact of churn on existing users.

There are tradeoffs to choosing any path, please attempt to identify them here.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?

```

```
