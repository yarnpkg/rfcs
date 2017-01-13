- Start Date: (1-10-17)
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Right now yarn does not have a concept of publishConfig. This setting already
exists in package.json for many npm packages. The setting allows you to set the
registry url where you want your package published.

# Motivation

The addition of picking up publishConfig from package.json will allow developers
creating internal packages to move across to using yarn with greater ease.

Developers will be able to use yarn publish to send their package to an internal
registry such as Nexus Repository Manager or Artifactory, while maintaining a
separate setting for the main registries such as npmjs or the yarn mirror. These
are very useful on their own for installing packages, but in many cases a
developer is going to want to publish their package to a different registry.

# Detailed design

To implement this I propose that publish.js be modified in such a way that it will override
registry settings if a package.json contains a publishConfig url. This will need to
modify seteps 2 and 3 of the process, particular the getToken (this occurs against a registry url)
and publish. I believe this can be accomplished by a if(pkg.publishConfig) and overriding
the registry url in config.

# How We Teach This

Since this is an existing npm feature, I believe not much will be needed
for users to understand or grasp this. Changes to documentation for yarn
publish are likely in order, but small changes to explain the behavior.

At least on our side (since this originates with Sonatype), we would likely
blog about this new behavior for our users so that they can adopt yarn
as well.

# Drawbacks

As with any new code, it's new code. Adding it expands the amount of functionality
that yarn now supports. That's the largest drawback I can think of. 

# Alternatives

I considered a --registry flag for the yarn publish command. This would likely accomplish the
same functionality but is prone to error as most hand typed things are. 

# Unresolved questions

I'm a relative newbie to yarn, so my design might be too simplistic or not accounting
for things I just don't know about. 
