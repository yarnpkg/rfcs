- Start Date: 2017-04-22
- RFC PR:
- Yarn Issue:

# Summary

Have `upgrade-interactive` run unit tests optionally to identify which upgrades
are safe to run.

# Motivation

It is difficult for a developer to know what packages they can safely upgrade
during an `upgrade-interactive` session. In many cases, a 'upgrade and pray' 
approach is the only current strategy. 

# Detailed design

For each available upgrade, Yarn should run the configured test hook `yarn test`
using the updated version of the package and report the results during an
`upgrade-interactive` session.

Current proposed design is a default behavior of using `yarn test` in the UI
which renders a checkmark or an X for each package update, using the latest version
of the package, to see if updating will cause tests to fail.

# How We Teach This

Minor changes to the documentation will be necessary, and a user-interface widget
in the `upgrade-interactive` session to indicate whether tests are passing or not
with this change.

Communicating the potential challenge of interaction effects (see Drawbacks) may be harder.

# Drawbacks

This has some constraints with respect to having to download the packages before
user interaction accepting them. It also requires a 'staging' environment for 
package selection which I am unfamiliar with whether yarn currently has.

This also has some challenges with respect to interaction effects. It is possible
that incrementing package A creates passing tests, and incrementing package B
creates passing tests, but simultaneously updating packages A and B causes
the tests to fail.

# Unresolved questions

- General engineering structure.
- How to handle the staging environment. 
- Whether the user should be prompted or have to specify a flag to use this feature.
