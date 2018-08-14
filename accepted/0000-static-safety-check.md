- Start Date: 2018-08-15
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Static safety-check for packages

# Motivation

The recent eslint-scope security incident has came as a surprise for many, if not all developers, as very few of us would expect a widely-used package on npm would be `hacked` in such a way.

Although measures must and have be taken in the publishing process both for eslint and npm team, human error and negligence are in its nature unavoidable; Thereof, for the developers to trust and embrace open source packages, actions must also be taken from the package manager's side, which developer interact everyday to download their packages, to provide an extra safety-net from malicious attackers.

The most concerning malware are the ones that steals data, thereof, we need to let developers know if the package they install have the ability to access the internet or not, and it should be up to the developers to decide if it is desired.

# Detailed design

The design is pretty straightforward:

Any network access would end up in requiring certain node modules or using certain browser api, therefore, it should not be difficult to do static analyses of those packages and provide basic safety check as for whether those packages requires those modules and/or calls those globals.

For packages that requires other packages, if any of their required package has `network capacity`, it should be marked as to have `network capacity`.

User need to manually confirm when they are adding a package with network capacity.

If the yarn.lock file indicates the package has no `network capacity` and yarn found that the currently pulled one does, it should throw an error.

In an CI/CD environment, the confirmation can be skipped if a lockfile is provided.

The result of such analysis can and should be cached on server. // it can and probably should be signed by the server of the package hash, analysis result and version.

When a publisher tries to publish a package with `network capacity` which previously doesn't, they will be asked to re-authenticate with their credentials, or provide a 2FA code. // not sure if yarn can do this. maybe only npm?

# How We Teach This

As this RFC improves existing API and extra warnings, the change should mostly be unobstrusive.

# Drawbacks

It may over-complicate the yarn package.

Those packages that uses network interface but only provides wrapper function are still considered package that has `network-capacity` and user would still be warned.

# Alternatives

The other designs might be to develop or use a separate package to perform static safety check and use that package within yarn to provide such functionality.

Or such measure could be and/or taken from the registry's side by providing an extra field in its response and relevant hash and signature from trusted server.

# Unresolved questions

Should we add extra endpoint to allow manual checking of existing access?

Perhaps we should let the user decide if they want this feature through config?
