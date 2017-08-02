- Start Date: 2017-08-02
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Add a "whitelist" feature that can be used in combination with `--ignore-scripts` to allow scripts only in certain packages.

# Motivation

Malicious install scripts in the NPM ecosystem has been a problem for years (`rimrafall` in 2015, [scripts stealing env vars in 2017](https://iamakulov.com/notes/npm-malicious-packages/))

The community could build a "blacklist" of known bad packages and refuse to install them or run their scripts, but that is very reactionary, not proactive.

One of the only ways to protect yourself is to run with `--ignore-scripts` enabled all the time, however there are valid packages that do need scripts to run (PhantomJS for example).

I propose a way to add a "white list" of packages that can still run scripts when `--ignore-scripts` is used.

# Detailed design

I envision either a field in `package.json` or in `.yarnrc` that is an array of package names. These packages would still have scripts run even if `--ignore-scripts` is specified.

# How We Teach This

Documentation around the config setting.

Possibly a blog post in response to this latest round of "OMG NPM is bad!" malicious packages news showing that Yarn is at least trying to do something to help.

(personally, my use case would be to just set ignore-scripts in my global yarnrc then opt-in individual packages with a whitelist only after I have manually verified that it looks valid)

# Drawbacks

Possible additional confusion if some scripts are still being run even though `--ignore-scripts` is passed.

# Alternatives

Do nothing.

Possibly add a different alternative to `--ignore-scripts` that more explicitely indicates that it would allow some whitelisted scripts to still run. `--ignore-some-scripts`? ;)

# Unresolved questions

* Where would a whitelist exist (`package.json` or `.npmrc`)
* What woud the setting name be.
