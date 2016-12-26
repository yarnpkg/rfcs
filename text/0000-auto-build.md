- Start Date: 2106-12-26
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Auto build on npm link/post install.

# Motivation

`git clone` yarn repo and `npm link` results in:

``` sh
$ yarn install
module.js:339
    throw err;
    ^

Error: Cannot find module '../lib-legacy/constants'
    at Function.Module._resolveFilename (module.js:337:15)
    at Function.Module._load (module.js:287:25)
    at Module.require (module.js:366:17)
    at require (module.js:385:17)
    at Object.<anonymous> (/Users/hemanth/labs/yarn/bin/yarn.js:9:17)
    at Module._compile (module.js:435:26)
    at Object.Module._extensions..js (module.js:442:10)
    at Module.load (module.js:356:32)
    at Function.Module._load (module.js:311:12)
    at Function.Module.runMain (module.js:467:10)
```

After 

```
npm run build
```

It works fine.


# Detailed design

It would be nice to have a post install, `run build` ?

# How We Teach This

Nothing new to teach here, it would just be a post install script.

# Drawbacks

Not that I can think of.

# Alternatives

None.

# Unresolved questions

Anything specific needed here? This is a follow up from [yarn#1416](https://github.com/yarnpkg/yarn/issues/1416)