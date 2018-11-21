- Start Date: 2018-11-21
- RFC PR:
- Yarn Issue: https://github.com/yarnpkg/yarn/issues/5773

# Summary

Yarn CLI command `yarn run` will be able to orchestrate various npm scripts, allowing them to run in parallel by default and matching any glob patterns.

# Motivation

Many build processes involve Grunt, Gulp, or other task runners that have great use cases but also require separate plugins, configuration files, and understanding of the API. `yarn run` would remove the need for installing these often global dpendencies and lower maintenance costs by providing a simpler interface defined in package.json. This would also cut down on the script run times and would be useful with workspaces as mentioned in https://github.com/yarnpkg/yarn/issues/6689.

Feature Requirements:

- Be able to run multiple commands in parallel across platform
- Show and differentiate console output for each command
- Compatible with existing behavior of `npm run` command

Prior Art:

- [npm-run-all](https://www.npmjs.com/package/npm-run-all)
- [gulp](https://gulpjs.com/)

# Detailed design

## Simple use case

// In `package.json`

```js
{
    "scripts": {
        "lint": "eslint .",
        "test": "jest"
    }
    "tasks": {
        "verify": ["lint", "test"]
    }
}
```

`yarn run verify` would run lint and test commands in parallel

## Advanced use case - pattern matching

```js
{
    "scripts": {
        "lint": "eslint .",
        "test:unit": "jest __tests__",
        "test:integration": "jest __integration__"
    }
    "tasks": {
        "verify": ["lint", "test**"]
    }
}
```

`yarn run verify` would run lint, test:unit, and test:integration commands in parallel

## Advanced use case - nested tasks

```js
{
    "scripts": {
        "build": "webpack",
        "lint": "eslint .",
        "test": "jest",
    }
    "tasks": {
        "verify": ["lint", "test"],
        "dev": ["verify", "build"]
    }
}
```

`yarn run dev` would run lint, test, and build in parallel

_NOTE_ Nested arrays or serial runs will not be supported in tasks because they can be done via scripts using `&&` operator

## Additional flags

`--hide-name`: By default, tasks will run by appending the name of the command to each line. Enabling this flag would allow it to be hidden from the output.

# How We Teach This

This will be an opt-in feature that would be compatible with existing behavior of `yarn/npm run`. However we should add documentation, as well as a simple migration guide.

# Drawbacks

- Increased complexity of yarn
- Feature not yet supported by npm
- Functionality has been partially implemented as a separate package by other open source projects

# Alternatives

`&` operator, which creates a sub-shell and would not allow modifying the output for readability
Third-party libraries, which are additional dependencies and can require complex command line args
Third-party task runners, which have issues mentioned above, mostly due to dependence on ecosystem/maintainers of plugins

# Unresolved questions

- Order of operations for pre/post hooks for installs if they are defined in tasks. Should they happen in parallel or serial?
