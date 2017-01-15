- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary
The Offline Mode, Same Packages, Network Performance Objectives must be applied global and symlink all libraries in the global flat node_modules folder.

The packages must not getaway from the nested node_modules. Instead extend the semver logic and use one flat node_modules using tar balls as follows.

# Motivation

When I create packages and applications, I cringe on the nested node_modules and number of copies getting created in each project folder. 

This is not efficient while development with say for example, angular2 using cli creates 100MB of node_modules alone.

This adds lot of time in creating projects and building projects. 

Also, there is not an efficient way of avoiding node_modules tree hierarchy hell.

# Detailed design

## Avoid Name collision using namespaces in Semver Extend packages and store as tar balls (gz, zip) all packages using the following format
* @angular-core-2.3.1-beta.internal1-lic-BSD-checksum.tar.gz
* @angular-core-2.3.1-beta.internal1-lic-LGPL-checksum.tar.gz
* @yarn-angular-core-1.0.0-@lic-COMM-checksum.tar.gz (This created because yarn customized angular-core in its own fork and it is being used

##Deep Fork
* Specify deep forks, aka fork from another fork need not be represented in the file name of tar ball. Instead, that must be represented in package.json


## Build using linking and bundling tar ball
* Create a tarball of vendor-bundle.js, vendor-dev-bundle.js and vendor-bundle-package.json with @yarn-yarn-vendor-tarball-checksum.tar.gz
* In project's package.json - keep the same with semver extension added.
* When building the project, if the vendor-bundle-package.json and vendor packages required from project's package.json are same then don't do any further build.
* Create a node_modules link folder to the tar ball file or extracted node_modules folder

# How We Teach This

Yarn is an improvement and an extension of how we install node_modules by providing ways to work offline. Yarn global link will take that to next step of how we can eliminate node_modules hierarncy tree hell from making the node_modules a flat structure in an intelligent way 

# Drawbacks

Build Time increases initially but that will be greatly negated by subsequent build of the same project or a different project or when creating new project within the same computer

Adds complexity to build time

Create a node_modules link folder to the tar bal file

# Alternatives
None

# Unresolved questions
None

Optional, but suggested for first drafts. What parts of the design are still
TBD?
None.
