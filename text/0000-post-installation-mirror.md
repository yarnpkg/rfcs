- Start Date: 2017-02-16
- RFC PR:
- Yarn Issue:

# Summary

Enable storing for post installation artifacts with Yarn offline mirror.

# Motivation

`Yarn` has improved node dependency resolution reliability dramatically.  However, we still can not
achieve truly deterministic dependency resolution and hermetic build without addressing issues with installation scripts.

Node modules allow arbitrary code execution before, during and after installation.  Although this feature has provided enormous flexibility for package owners, it is also one of the root causes for non-deterministic build.  Some packages use installation scripts to download additional codes and / or dependencies. This means those packages cannot be installed in network restricted environments (NRE). Production zones in enterprise environments are one of the examples of those NREs. They tend to restrict network access for obvious security reasons.

Without further discussion with validity of installation scripts, a possible workaround can be achieved by storing the post installation artifacts using yarn offline mirror. In essence, we are aiming to provide a better alternative than an already prevalent practice among enterprise users, namely, save or check in the post installation node_modules folder.  Formalize this practice provides at least following benefits:
- Allows node build to be truly offline, deterministic and hermetic
- Allows package owners to continue use installation scripts
- Allows saving of a single .tar.gz file instead of hundreds, if not thousands of files
- Easier code review for addition, update and deletion of dependencies

# Detailed design

## Assumptions
- Most, if not all, installation scripts only modify files and directories within its own folders.
- Storing post installation artifacts is voluntary on per package basis.

## Analysis of installation scripts
It is difficult to categorized all possible usages of installation scripts as they are truly free formed. That being said, there are a couple of common patterns have been observed so far:
- Download additional dependencies or codes
- Compile node native extensions

This observation leads to the assumption that most installation scripts only modify files and directories within its own folders.  Consequently, we can store the post installation content as artifacts without worrying about inter-module dependencies.

##Modification to `yarn` offline mirror structure and `yarn.lock`

- Store post installation artifacts under post-installation subdirectory when using yarn offline mirror.  _resolved_ field should reflect this change as well.

examples:
```
node-pre-gyp@^0.6.29:
  version "0.6.31"
  resolved "post-installation/node-pre-gyp-0.6.31.tgz#sha1"
  dependencies:
    mkdirp "~0.5.1"
    nopt "~3.0.6"
    npmlog "^4.0.0"
    ...
```

##Modification to cli

Storing post installation artifacts should be a voluntary feature.  This means that we should only store post installation artifacts when asked specifically.  When installing from offline mirror, post installation artifacts should be given preference since the existence of such artifacts reflects specific actions taken by the maintainer of the offline mirror previously.

- For _add_ command, add an additional command line parameter --save-post-install. When this parameter is specified, store post installation artifact to offline mirror. The post install artifact should include all files and directories after running installation scripts, but exclude node_modules subdirectory.

- For _install_ command, search post install subdirectory first, if there is no post installation artifacts, fail through to existing work flow.

- When installation a post install artifact from offline mirror
  - extract the tar.gz file in place
  - do not run install scripts
  - install dependencies and create bin link as usual

# How We Teach This

*What names and terminology work best for these concepts and why?*
"post installation artifacts" to distinguish from node modules
"network restricted environments (NRE)" where access to Internet is controlled
"hermetic builds" means all dependencies are included, could be used as synonym for "deterministic builds"

*How is this idea best presented?*

- Emphases on the concept of "hermetic builds".
- Emphases on `yarn` usage within "network restricted environments (NRE)".

*Would the acceptance of this proposal mean the Yarn documentation must be re-organized or altered?*

No.

*Does it change how Yarn is taught to new users at any level?*

This feature should be considered as advanced and should be taught specifically to following users:
- users who need to operate within network restricted environments (NRE)
- users who value deterministic dependency resolution and hermetic builds above convenience

*How should this feature be introduced and taught to existing Yarn users?*
Explain the intended use case with illustrated work flow.

# Drawbacks

- Somewhat a deviation from current node community norm.  This feature may potentially require additional teaching.
- Added complexity for installation work flow and offline mirror structure
- Depending on directory structure to identify post installation modules

The following points are drawbacks under certain circumstances and advantages for other circumstances.

- Native extension support
If a node module has native extensions, a stored post installation module will not work on platforms different from where the module is created.

Currently, native extensions are compiled during installation. However, since the compiler and libraries are provided by running environment, the compilation output are not guaranteed to be repeatable. To ensure hermetic and repeatable build, a separate RFC is necessary due to the complexity of supporting node native extensions .

- Lost of some flexibility
In at least one package (cldr-data), [build result can be altered by environment variables $CLDR_COVERAGE](https://github.com/rxaviers/cldr-data-npm/blob/master/install.js#L91).  Caching post installation artifacts will lose this flexibility.

- Update to installation time downloads are ignored / require explicit action
Installation scripts tends to download the latest version of dependencies.  A stored post installation artifacts will always have the same version of dependencies and thus potentially will not have the latest dependencies.  To update such installation time downloaded dependencies, explicit actions from offline mirror maintainers will be required.

# Alternatives

The following alternatives have been considered:

- Do nothing
One may argue that this problem is not worthy to be addressed and do nothing is the correct approach. It has been in existence since the inception of npm and the node community has thrived at the same time. A second argument is that users in network restricted environments (NRE) are not the intended customers.
However, as adoption of node widens, the inability to run node build in network restricted environments has been and will continue to be a hurdle for adoption.  Not addressing this problem is no longer a valid option.

- Work with each individual package owners to make sure package installation is hermetic
There are several drawbacks of this approach:
  - Installation scripts may serve legitimate purposes in certain circumstances
  - Requires significant efforts to educate node module writers
  - Working on a per package basis and updating all dependent packages might take a long time for the necessary changes to propagate.

# Unresolved questions


