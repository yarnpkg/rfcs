- Start Date: 2017-01-16
- RFC PR: (leave this empty)
- Yarn Issue: https://github.com/yarnpkg/yarn/issues/2464

# Summary

Torrenting dependencies should be more efficient and faster than downloading them from 1 or 2 registries, and it would reduce the load on those registries. The hash of each dependency from the `yarn.lock` file would be used as the torrent `infoHash`. If `yarn` users seeded their local registries by default, there would be a large group of peers to download from. [WebTorrent](https://webtorrent.io/) is a pure JS implementation of a torrent client, which could be integrated into `yarn`, potentially as another type of registry.

# Motivation

Faster, more efficient downloads of dependencies and reduced load on the npm and yarn registries.

# Detailed design

WebTorrent could be used to implement another type of registry. Peers would be queried for the hash of the dependency taken from the `yarn.lock` file. This feature would only work with `yarn.lock` files, not with `package.json` files (which could create yet another incentive to swtich to yarn).

When installing yarn maybe you would be asked if you are okay acting as a torrent seed for the dependencies you have installed. It might be useful to add some commands to the yarn CLI like `yarn torrent start --upload-bandwidth=50`, `yarn torrent stop`.

# How We Teach This

Torrents are widely known so it would probably be sufficient to explain that yarn can be used to seed/fetch dependencies from peers using a JS implementation of a torrent client.

# Drawbacks

People might not want to have yarn seed their local copies of dependencies by default. If seeding were not turned on by default there might not be enough seeds to make torrent downloads faster than the regular ones.

An attacker might be able to use a sha1 hash collision to make peers download a malicious version of a dependency.

A vulnerability in the WebTorrent client might pose a security risk if a malicious seeder could send an exploit over their WebRTC connection.

# Alternatives

Continuing to download all dependencies from a small number of registries.

# Unresolved questions

- Should yarn seed local dependencies by default?
- Does WebTorrent have a way to limit the upload bandwidth or just the [number of connections](https://github.com/feross/webtorrent/blob/master/docs/api.md#client--new-webtorrentopts)?

