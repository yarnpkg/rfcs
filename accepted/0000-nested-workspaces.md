- Start Date: 2018-08-17
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Add an ability to use nested workspaces.

# Motivation

In some cases, it's needed to use packages with workspaces inside another package.
For example, we have some independent projects and another project which contains integration tests.
The possible structure can look like this:

```
Integration_tests_package
|-- package.json # has workspaces
|-- projects
|   |-- Project_A
|   |   |-- package.json # also has workspaces
|   |   |-- packages
|   |   |   |-- aa
|   |   |   |-- ab
|   |-- Project_B
…
```

# Detailed design

Nested workspaces can be implemented by a small change in the method responsible of finding a workspace root.
Current implementation finds the closest `package.json` which has a workspaces definition with the current project in it.
Instead, we need to find root `package.json` with workspaces.

# Drawbacks

This feature can affect some of the existing projects if they have a workspaces definition in one of the ancestor folders.

# Unresolved questions

Actually, there are two ways of implementation: using a definition of workspaces from parent package or merging all definitions.
The first way is more preferable because it is more transparent and gives more control but it ignores all information about nested workspaces.
On the other hand, the second way uses already defined workspaces from nested packages but it can be undesirable in case we want to override some workspaces in the parent project.
