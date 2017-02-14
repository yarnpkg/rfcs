- Start Date: 2017-02-10
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Show only updated packages on `yarn upgrade` with the new version.

# Motivation

When updating I want to know which of "my" packages have been updated (not dependencies of my dependencies) so that I can verify a package has been updated to the expected version. This makes it easy to spot incorrect version constraints you define within your package.json.

# Detailed design

Instead of just showing all dependencies the following should be shown.

```
success Saved lockfile.    
success Saved 774 new dependencies.    

Updated direct dependencies:    
├─ @webcomponents/custom-elements@1.0.0-alpha.3    
├─ @webcomponents/shadycss@0.0.1    
├─ @webcomponents/shadydom@0.0.1    

All updated dependencies:    
├─ @webcomponents/custom-elements@1.0.0-alpha.3    
├─ @webcomponents/shadycss@0.0.1    
├─ @webcomponents/shadydom@0.0.1    
├─ abbrev@1.0.9    
├─ accepts@1.3.3    
├─ acorn-jsx@3.0.1    
├─ acorn@4.0.11    
├─ ajv-keywords@1.5.1    
├─ ajv@4.11.2    
```

# How We Teach This

The new section should be shown when running upgrade. Nothing else should be needed.

# Drawbacks

Your dependencies will be shown twice, or a `flag` needs to be introduces or the current view (showing all packages is not available / needs a flag).

# Alternatives

- add a flag like `--only-deps` to only show my dependencies
- or add a flag like `--all` to show the current version

# Unresolved questions

-
