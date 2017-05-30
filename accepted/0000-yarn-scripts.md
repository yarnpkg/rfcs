# A new system for defining scripts + “yr”

* Start Date: 2017-05-19
* RFC PR:
* Yarn Issue:

# Summary

Defining scripts in a single JSON in package.json files is cumbersome.

# Motivation

Running scripts through yarn is one of the most common interactions with Yarn next to managing dependencies. The standard way of defining scripts is through the `scripts` field in `package.json`. Because this file is encoded in JSON and because of the format that was chosen, defining more complex scripts is difficult: [Example package.json](https://github.com/facebook/jest/blob/b72cd6c95335151e723c0a4b57273bce0e519630/package.json#L53-L74).

This RFC aims to solve this by introducing a new way to configure scripts using YAML. Many projects already use YAML to configure CI systems which is solving a similar use-case: running a sequence of scripts. [Example travis.yml](https://github.com/facebook/jest/blob/master/.travis.yml). To keep backwards compatibility, if the scripts definition file is absent, Yarn will fall back to the current behavior in `package.json`. To keep compatibility with other parts of the JavaScript ecosystem, such as npm, this RFC proposes to exclude script names related to installs and as such is purely meant to improve developer experience.

# Detailed design

* Change in behavior for `yarn run <script>`:
    * If a `yarn-scripts.yml` file exists, read and parse the file and see if there is a `<script>` matching that.
    * If a script is matching, run it, otherwise default to the previous behavior of looking at `package.json` scripts or binaries in `node_modules/.bin`.
* A few scripts like install are blacklisted to ensure ecosystem compatibility. Yarn will throw if a blacklisted script is defined in a `yarn-scripts.yml` file.
    * *TODO: full list of commands to be blacklisted.*
* If neither a `package.json` or `yarn-scripts.yml` exist in the current directory, Yarn will now go up the file-tree until it finds either file and runs the scripts specified.
* Yarn will start shipping with a binary `yr` that forwards to `yarn run` as a shortcut.
    * If `yr` is executed with no arguments, it will implicitly call `yr start`.

File format:

```
clean: rm -rf ./packages/*/build

lint:
 - yr eslint
 - yr lint-docs

test:
 - jest
 - yr lint
```

Script execution:

* A list of scripts is executed sequentially. If one of them fails, bail and fail the process.

# How We Teach This

* Documentation.
* Blog post.

# Drawbacks

Implementing this feature is backwards compatible and won't affect existing users.

# Alternatives

Prior art: [nps](https://github.com/kentcdodds/nps).

# Unresolved questions

* Should there be an init/conversion script that converts the `scripts` field to a file?
* **nps** ships with support for a script file encoded in JavaScript. Shall we consider this as well?
* ?
