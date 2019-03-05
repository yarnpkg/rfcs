- Start Date: 3/5/19
- RFC PR:
- Yarn Issue:

# Summary

Add a flag to the CLI that allows users to set the working directory from which the subsequent command is executed.

# Motivation

In projects where _package.json_, _yarn.lock_, and _node_modules_ do not live at the root level, users must change directories before running yarn commands like `install`. This becomes repetitive and prone to human error, especially when use in CI/CD configs, containerization, etc. This is partially supported today with the ability to specify the modules folder; however, you still need to have a dummy/proxy _package.json_ at your project's root and the yarn.lock file is generated at the root, not in the desired subfolder. The desired outcome is the ability to set the working directory via a CLI flag and subsequently, the _.yarnrc_ config, thus allowing users to run commands and custom scripts from a parent or any other directory. Consider the following project structure:

* client  
 * package.json  
 * node_modules  
 * yarn.lock  

On the surface, running `cd client && yarn install && yarn customscript` from the project root seems relatively painless. That said, configuring CI/CD workflows that revert to the root directory between build steps, working within bash in the volume of a container, etc. typically results in the repetition of the `cd client` portion of that commands. Additionally, this can be confusing to inexperienced engineers and requiring users to constantly change directories is prone to human error. The implementation of this new feature may also negate the need for the `--modules-folder` flag and other settings designed to give users the freedom to use non-traditional project structures.

# Detailed design

Running `yarn --working-directory=client install` from a project's root, where _package.json_ lives at '/client/package.json' (and not in the root), would generate both _node_modules_ and _yarn.lock_ in the 'client' directory along side the existing _package.json_ file. Additionally, running commands like `yarn --working-directory=client customscript` would succeed, as the all required files/folders exist at the location provided. As an added convenience, users could forgo appending the flag to every command prior to execution if they provide a _.yarnrc_ file at the project root with the following contents:

```
--working-directory "client"
```  

# How We Teach This

The new CLI flag can be introduced via normal channels, e.g. the CLI page in the documentation, release notes/changelogs, etc. as done traditionally with other new configuration options.

# Drawbacks

Potential draw backs with semantics, i.e. using 'directory' alongside 'folder' may be confusing to some. The term 'working directory' can take on different meaning depending on your development environment, perhaps a name with the word 'root' would better serve the users. Their may be potential security risks with allowing absolute paths; as a precaution, perhaps only relative paths should be allowed. 

# Alternatives

By not solving for this use case, Yarn will be indirectly opinionated about project structure, which seems counter to the mission. As time passes and JavaScript and other frameworks progress, the community at large is embracing alternative and complex project structures which don't always support root-based configuration and execution. Giving users the freedom to organize their project to suit their needs while maintaining the ease of running yarn commands from any directory is a must have. The mechanism described above is merely a suggested approach, alternatives of merit are certainly welcome. Also considered was simply expanding on the `--modules-folder` flag by also supporting `--lockfile-folder` and `--package-json-folder` options; clearly, this is less succinct.

# Unresolved questions

N/A
