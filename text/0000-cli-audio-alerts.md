- Start Date: 2017-04-28
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary
Add an option to the CLI that turns on alert sounds for Yarn completion, warning, and error events.

# Motivation
In short, this RFC is motivated by situations such as this:

    shia@labeouf ~> yarn run develop

    yarn run v0.23.2
    $ watch 'yarn run build' src/ -ud 
    > Watching src/

    yarn run v0.23.2
    ✨  Done in 0.76s.

    yarn run v0.23.2
    $ rollup
    ✨  Done in 3.18s.

    yarn run v0.23.2
    $ rollup -c -i src/content/main.js -o dist/content.js -f iife -n content 
    ✨  Done in 2.04s.

    yarn run v0.23.2
    $ rollup -c -i src/content/main.js -o dist/content.js -f iife -n content 
    🚨   (babel plugin) src/main.js: Unexpected token, expected , (31:27)
    undefined (31:27)

    error Command failed with exit code 1.
    info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.

During development, Yarn often runs as a background process, hidden away in another console tab or window, or not visible at all. The error above would then occur silently, unnoticed. This can be very frustrating when we modify and save code while a persistent build–test process is expected to be running: we then expect our code to build and run automatically. If we accidentally type a syntax error, it can take quite some time until we notice that our build is no longer updating. Alternatively, we might keep the possibility of a silently failing build in our mind at all times, switching back and forth between code and build process repeatedly and haphazardly. Either way, it’s an unnecessary cognitive burden, distracting and interrupting our actual work.

Visual or aural feedback when a build succeeds or when an error occurs solves this problem, and many build systems provide affordances for them. Gulp, for instance, uses Node streams to capture error events, allowing packages like [gulp-notify](https://www.npmjs.com/package/gulp-notify) to play sound alerts when errors occur. In the world of Clojure, the Boot build system even has a similar `notify` task built in and already configured.

The Yarn team should consider building audio alerts into Yarn for usability.
One might think of them like emoji ✨  for your ears.

# Detailed design
`yarn` would support a ew flag, `-a, --alert [types]`, for all its commands. This flag would modify Yarn’s reporting such that, when its command finishes, it would emit sounds when certain events (finishing, warnings, and errors) occur.

`[types]` would be a space-delimited unordered list of zero to three of: `finished`, `warning`, and `error`. By default, `--alert` includes all three levels. For instance:

* `yarn run develop --alert finished error` would play finished and error sounds.
* `yarn run develop --alert finished error warning` would play finished sounds, error sounds, and warning sounds.
* `yarn run develop --alert` would be equivalent to including all three alert types.

Depending on the team’s preferences, the sounds may either be MP3 files included in Yarn or synthetic sounds procedurally generated from code.

# How We Teach This
This RFC introduces new terms—“alerts” and “alert types”. It also makes more explicit the event types that already occur in Yarn’s reporters (`finished`, `warning`, `error`). It would not affect much else in Yarn’s design or documentation, being orthogonal to aspects other than reporting. It may also be worth introducing early in tutorials, since it might have a “wow” factor with little clutter (`yarn -a`).

# Drawbacks
Deferring audio alerts to separate projects (see Alternatives) would keep Yarn’s core cleaner. Building audio alerts into Yarn has the following disadvantages:

1. More dependencies (e.g., [play-sound](https://www.npmjs.com/package/play-sound), [baudio](https://www.npmjs.com/package/baudio) would have to be added to Yarn.
2. Additional code would have to be added to Yarn.
3. First-time users would have one more setting to worry about.
4. If MP3 files are used (instead of procedurally generated waveforms), those MP3 files may moderately increase Yarn’s size. Depending on the library chosen for their playback, they may also bring in codec-license complications.

Disadvantages to not building them in include:

1. Users would have to verbosely enter something like `yarn --json [command stuff] | yarn-alert` every time we wish to use `yarn` with alerts. (Or the project could include yet another CLI command wrapping `yarn`…just like how `yarn` is designed to supplant the `npm` command. It would be quite redundant.)
2. Its omission from the core would make it much less likely that it would be discovered and used by Yarn’s users.

This functionality would be nearly universally useful: certainly at least as useful as console emoji. As the number of users they can help is large, the negative impact of excluding them—and the positive impact of including them—are thus correspondingly large.

# Alternatives

## Alternatives to including alerts in core
Yarn supports a [JSON reporter](https://github.com/yarnpkg/yarn/blob/master/__tests__/reporters/__snapshots__/json-reporter.js.snap), which is undocumented online but accessible using the `--json` flag. This reporter causes Yarn to output a stream of JSON objects to standard output, each object representing an event that would have otherwise been transformed into a human-readable message complete with emoji.

One alternative (and not mutually exclusive) design to this RFC would be to create a flag that pipes raw JSON output to a command *in parallel to* Yarn’s standard human-readable console output. Such a flag would allow anyone to extend Yarn without interfering with its standard console reporting. Someone could, for instance, create a separate project project that would parse Yarn’s JSON output and emit sounds whenever it detects certain types of events.

Another (not mutually exclusive) alternative would be to separate the functionality of Yarn’s `ConsoleReporter` into its own library, one which other libraries could use to emulate human-readable Yarn console reporting.

One third alternative would be to instead add a new command to Yarn—call it `yarn report`—that accepts Yarn JSON events from standard input and converts them into human-readable console reports.

These alternatives are not mutually exclusive with this RFC. They may deserve their own parallel RFCs. However, implementing any of them would make this RFC unnecessary in a strict sense, though perhaps not in an end-user, ergonomic sense.

## Alternatives in sound types and playback methods
**TBD**: [play-sound](https://www.npmjs.com/package/play-sound) vs. [baudio](https://www.npmjs.com/package/baudio) vs. etc.

## Alternatives in CLI options
**TBD**: Hierarchial/numeric alert levels

# Unanswered questions
See Alternatives subsections. Also:

* Should progress-event alerts also be supported? Any other event types?
* Is it actually impossible to output both JSON and console reporting in parallel without reimplementing `ConsoleReporter`?
