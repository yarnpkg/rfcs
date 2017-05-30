- Start Date: 2016-10-11
- RFC PR:
- Yarn Issue:

# Summary

Add the ability to install from and authenticate against an alternative
registry, via the environment variables `YARN_AUTH_TOKEN`, `YARN_REGISTRY`,
and the `.yarnrc` settings `registry`, and `token`.

# Motivation

Services like npm, Artifactory, and Sinopia allow users to develop private
JavaScript dependencies using the same tools and workflows used for open-source development. To be able to install from these alternative registries, yarn needs to be able to provide an authentication token and to be able to point to an alternative
registry URL. To be able to support private dependencies in a CI environment, it must be possible to provide authentication without manual intervention, e.g., having a user login before each authenticated install.

It is becoming a popular practice in the JavaScript world to automatically publish
a package once builds pass in CI, see ([Travis CI Deployment](https://docs.travis-ci.com/user/deployment),
[semantic-release](https://github.com/semantic-release/semantic-release)).
To facilitate this, it would be useful to be able to publish from the yarn
client without needing to manually authenticate.

Supporting both these use-cases would allow JavaScript developers to more easily use Yarn to
develop private using private dependencies in a CI environment.

# Detailed design

## authenticating JSON requests

If the upstream JSON meta-information request is being made to a registry
matching `YARN_REGISTRY`, or `registry` in `.yarnrc`, then the
authentication header would be populated with the corresponding `YARN_TOKEN`, or `token` in `.yarnrc`.

## authenticating tarball requests

If their base URL matches the URL configured in `YARN_REGISTRY`, the requests for upstream tarballs would also have the authentication header populated.

## mitigating publication security concerns

In order to prevent a publication attack, such as the attack discussed in [this article](http://www.infoworld.com/article/3048526/security/nodejs-alert-google-engineer-finds-flaw-in-npm-scripts.html), we could introduce an additional configuration variable, e.g., `YARN_REAUTH_PUBLISHES`/`auth_publishes`, which would require that users re-enter credentials
to publish a module; file permissions could be used to prevent lifecycle-scripts
from updating this setting in `.yarnrc`.

# How We Teach This

## Glossary of Terms

* **auth_token**: the opaque authentication token provided by npm (and other registry
   implementations) after a successful login.
* **registry**: the remote location hosting package meta-information and package tarballs.
* **private package**: a package stored on a remote registry, that requies authentication
  to install it.
* **publication**: the process of publishing either a public or private package to a registry.
  Requires authentication in both use-cases.

## Add section to documentation about installing private dependencies

A section should be added to the yarn documentation, discussing how to add an
authentication token to `.yarnrc` and how to configure yarn to speak to
alternative registries.

## Add section to documentation about using yarn in a CI environment

It would be worth adding an additional section to the yarn documentation,
discussing how to configure a few common CI environments, e.g., Travis CI. It has
been my experience that the process of auto-generating configuration, for instance an
appropriate `.yarnrc` file, often trips people up.

## Add section to documentation about security

A section should be added to documentation discussing the security concerns surrounding
headless publication; with some approaches to mitigating these concerns. And a discussion
of the importance of keeping your `auth_token` secret.

# Drawbacks

## security!

There are inarguably some security concerns with giving yarn the ability to install
and publish modules in a headless CI environment. I believe this concern can be
partially mitigated by introducing a setting like `reauth_publishes`, and with
appropriate configuration of the read/write permissions on the local `.yarnrc` file.

The default behavior could require authentication for publishes, while providing
the ability for a CI environment to perform a headless publication; it's worth noting that sandboxed CI environments are less at risk to a worm-attack than a user's local
machine, which tends to have a large number of repositories cloned to it.

## no support for multiple registries

In the current `.npmrc` file format, multiple registries and API tokens can be configured:

```
//32.237.243.48:8081/:_authToken=e8c0bbbe-aaaa-4b59-8275-8784a592afaa
//44.237.243.48:8080/:_authToken=29958a1f-bbbb-4f6f-9d37-7e61676293bb
//99.86.144.137:8080/:_authToken=7d3a257b-2222-4d20-b43a-7ca2f58b2299
```

The implementation proposed in this document would only support one registry/token.

# Alternatives

## support headless private installs but not publications

To mitigate the threat of a worm we could support headless authenticated installs,
while continuing to always require that a user authenticate during publication.

## support `.yarnrc` configuration but not environment variables

It's easier to prevent lifecycle scripts from writing to `.yarnrc`, then it would be
to prevent lifecycle scripts from reading from and populating environment variables --
perhaps we support configuration in `.yarnrc`, but not environment variables.

## handle multiple registries and scopes

Perhaps rather than building an MVP that supports only a single registry with an
authentication token, we should support the more complex use-case handled by `.npmrc`:

* an authentication token can be provided for each registry URL.
* multiple scopes (`@mycompany`) can be associated with multiple registries.
