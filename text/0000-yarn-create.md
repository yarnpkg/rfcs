- Start Date: 2017-03-31
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

The idea would be to make it easier for users to setup projects from scratch.

# Motivation

During the past few years, we've seen an increase of the number of "boilerplate" projects, each one aiming to lower the complexity of creating new projects with Javascript. Create React App is a good example, but we can also mention Neutrino, released by the Mozilla teams, or Next.js, which recently reached its v2 milestone.

Despite their utility, using these tools still require to manually install them, and then to keep them updated. A common workaround has been to prone the use of a "core" package to be installed locally, and a "cli" package to be installed globally that would act as a bridge to the core package. It feels like a hack, and maybe we could do something to help both project maintainers their users.

# Detailed design

This RFC suggests to add a new command:

```
$> yarn create <package-name>
```

Running this command would have the samne effect as:

```
$> yarn add --dev <package-name>
$> cd node_modules/<package-name>
$> npm_create_path="$OLDPWD" yarn run create
```

One could assume that a simple boilerplate would be configured as such:

```
{
  "scripts": {
    "create": "cp -rf ./files $npm_create_path"
  }
}
```

This RFC doesn't cover the case where `yarn create` is called in an already existing package without argument - it is suggested that the boilerplate modules register new script commands that the user could then use:

```
$> cat package.json
{
  "scripts": {
    "cra": "create-react-app"
  }
}
$> yarn cra eject
```

# Alternatives

  - We could do more than just running a script (maybe automatically copying files, etc), but I'm not sure it would be a good idea - I feel like such a feature should remain very simple.

  - The `$npm_create_path` variable could be an argument instead of an environment variable. However, it would then be harder to use shellscripts as `create` scripts, and would make programming errors more potent (for example, a `create` script that would end with an `rm -rf` would be bad).

  - The package could be added globally instead of locally, but doing so could cause versioning issues: if we choose to upgrade `<package-name>` when running `yarn create`, then the other projects created before this time would no longer be able to use the previous version. If we choose not to upgrade `<package-name>`, then the users will probably never upgrade at all.

  - The script could be named differently. However, "create" isn't currently used as as lifecycle hook, and doesn't see a lot of usage (of the 490,000+ packages on the npm registry, only 33 of them have a script called "create").

# Unresolved questions

  - The best way this feature could be implemented would probably be via a plugin, since the core project would then not have to bother about cluttering. Unfortunately, we've not yet reached the point where we can start exposing a public API, and as such it seems difficult to avoid adding the command into the core app right now.

  - Should the extra arguments be forwarded to the create script? If it works like the regular ones then no, and users would have to type `yarn create <package-name> -- --no-eslint` instead of `yarn create <package-name> --no-eslint`. However, fixing this behaviour might require a different RFC, since consistency would suggest to fix how the parameters are passed to the scripts as well.
