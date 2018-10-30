- Start Date: 30 Oct 2018
- RFC PR: https://github.com/yarnpkg/rfcs/pull/106
- Yarn Issue: n/a

# Remove `yarn check`

## 1. Motivation

Running `yarn install` should work out of the box. There's no reason for `yarn check` to exist from a user perspective - they should never have to run it, because installs must always be right.

Being not particularly useful, the `yarn check` command receives less attention and as a result it often yields wrong / confusing results. This leads users to believe that `yarn install` is broken when it's in fact `yarn check` (https://twitter.com/betaorbust/status/1055610508533878784).

## 2. Detailed Design

We'll remove the `yarn check` command in Yarn 2.0.

## 3. How Can We Teach This

This command will be marked as deprecated, and running it will exit with an error code and an error message explaining why it got removed.

## 4. Drawbacks

- The one use of `yarn check` is to provide debug information when installs aren't properly made. In practice it's never used this way, even by maintainers.

## 5. Alternatives

- We could fix `yarn check`. This would prove time consuming, of little value because of the reasons detailed above, and resources would be better spent on other worksites.
