- Start Date: 2016-10-11
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

Yarn must have a Node.js compatibility story which enables it to be used in enterprises and side projects everywhere. 

# Motivation

A package manager must have an incredibly detailed support plan for the community which it serves. It must do this as the foundation for building tools within the ecosystem. This support plan will likely need to _exceed_ the [support guarantees of the underlying ecosystem](https://github.com/nodejs/LTS).

# Detailed design

The current major version of Yarn will work with any Node.js version which has ceased being supported within the last year. One year after the Node.js LTS Working Group's cease of support announcement for any particular major version Yarn will no longer explicitly support that version of Node.js. It will then remove that version from its automated tests.

Yarn may drop support in a _patch_ update per SemVer.

# How We Teach This

The answer is simply communication. This should appear in blog posts, tweets, and documentation on the website. The Yarn team should participate in the Node.js LTS WG.  

# Drawbacks

The costs of an extended support model are borne by those who maintain the project. This takes a tremendous amount of effort and discipline and means that we don't get the newest features of JavaScript. This is a tradeoff that we must make as a consequence of being a foundational tool.

# Alternatives

As an alternative we could adopt a policy which explicitly matches the Node.js LTS policy. This maps to [other Node.js ecosystem projects such as Ember](http://emberjs.com/blog/2016/09/07/ember-node-lts-support.html).

Not setting a strategy is not a viable option when we have enterprises which need to be able to follow a known predictable schedule for their updates. 

# Unresolved questions

How much appetite is there in to maintain this project in legacy versions of Node.js?