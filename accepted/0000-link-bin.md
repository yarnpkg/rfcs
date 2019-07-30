- Start Date: 2019-07-30
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

The new `dependenciesMeta[dependency name].linkBin` field allows to prevent a dependency from
linking its bin files to `node_modules/.bin`.

# Motivation

Sometimes several dependencies are exporting bin files that have the same name.
For instance, both `gulp` and `gulp-cli` implement the `gulp` command.
Which dependency's command should be linked into `node_modules/.bin`?
There is currently no way to choose which dependency's command will be linked,
when there's a conflict.

# Detailed design

Yarn already has a `dependenciesMeta` field. We can add a new field called `linkBin`.
When `linkBin` will be `false`, the dependency's commands will be not linked.
For instance:

```json
{
  "dependencies": {
    "gulp": "4.0.0",
    "gulp-cli": "2.2.0"
  },
  "dependenciesMeta": {
    "gulp": {
      "linkBin": false
    }
  }
}
```

# How We Teach This

During a bin name conflict, Yarn may print a warning and suggest users to
declare a dependenciesMeta field with `linkBin: false` for one of the conflicting dependencies.

# Drawbacks

# Alternatives

# Unresolved questions

The suggested solution might be not flexible enough.

Should it be possible to disable specific commands only?
Should it be possible to rename commands?
