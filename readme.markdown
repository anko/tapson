# Tapson specification

Tapson is a simple protocol for communicating the results of software tests in
a language-agnostic manner.  It is inspired by the [Test Anything Protocol
(TAP)][1] but is simpler, uses [JSON][2], and supports streaming, parallel
tests.

## In summary

`\n`-delimited JSON objects.

An object that contains an `ok` property is a *test result*.  One that doesn't
is a *test plan* and must contain an `id` that a later test can match.

The `expected` property tells a human what a test plan expects to see.  The
`actual` property tells a human what actually happened in a test result.  Both
are optional.

## Why

There's no equivalent existing protocol.

The closest is [Test Anything Protocol][3], which tapson has advantages over:

-   Tapson distinguishes between *planning* to run a test and *actually running
    it*, which enables parallelism without mix-ups.  (TAP assumes tests execute
    immediately.)
-   Tapson is simple.  (TAP is weighed down by features like YAML blocks,
    "TODO" annotations, diagnostic messages, test-skipping and bailout
    directives.)
-   Tapson is JSON-based.  (TAP uses an informally specified custom format.)
-   Tapson is better specified.  (TAP implementations have diverged due to the
    vague spec, which partly defeats its purpose.)

## Examples

### Simple tests

**Input**:

    {"ok":true}
    {"ok":true}
    {"ok":false}

**Interpretation**:  Three tests ran, one failed.  No further details were
given.

### Optional Explanatory properties

**Input**:

    {"ok":true, "expected":"Calculator set up works"}
    {"ok":false, "expected":"Negative number support", "actual":"Only unsigned int is supported"}
    {"ok":false}

**Interpretation**:  Three tests ran, one passed, two failed.  The first tested
calculator set up.  The second tested for negative number support and failed,
finding only unsigned integers.  The third test failed too, but further details
were not specified.

### Mixed planned test and immediate tests

**Input**:

    {"id":"55ca6286e3e4f4fba5d0448333fa99fc5a404a73", "expected":"Page loads"}
    {"ok":true, "expected":"Simple arithmetic works"}
    {"ok":true, "id":"55ca6286e3e4f4fba5d0448333fa99fc5a404a73", "actual":"200 OK"}

**Interpretation**:  Two tests ran, both succeeded.  The first was planned with
an identifier and expected value, and its result arrived later with a matching
identifier.  The other test was not planned with an identifier, but succeeded.

### Multiple planned tests, run asynchronously

**Input**:

    {"id":"c7059bb19433cc3cabaa6236c83d56668a843dd2"}
    {"id":"7bbef45b3bc70855010e02460717643125c3beca"}
    {"id":"7bbef45b3bc70855010e02460717643125c3beca","ok":true}
    {"id":"1e7720a3460b8a84ac4ba27880d64526a3872f1c"}
    {"id":"1e7720a3460b8a84ac4ba27880d64526a3872f1c","ok":true}
    {"id":"c7059bb19433cc3cabaa6236c83d56668a843dd2","ok":true}

**Interpretation**:  Three tests were planned, and all succeeded.  (Note that
the tests were planned in a different order to how their results were
received.)

### Unconfirmed planned test

**Input**:

    {"id":"4a4121ecd766ed16943a0c7b54c18f743e90c3f6"}
    {"id":"4430bb02f6ed700d4408eb307b25f8b1a25d93de"}
    {"id":"4430bb02f6ed700d4408eb307b25f8b1a25d93de","ok":true}

**Interpretation**:  Two tests were planned.  The first failed with no
explanation, the second succeeded.  (Note how the first planned test received
no result; it is treated as failed with no further explanation.)

### Test result with no matching plan

**Input**:

    {"id":"688e71a079f58305c76005ddd2711db9088ed105", "ok":true}

**Interpretation**:  One test ran and passed with no further explanation.
(Note how the test is treated as if it had no identifier when a corresponding
plan was not present.)

### Object with additional unspecified properties

**Input**:

    {"ok":true, "weaponOfChoice":"luftwaffles"}

**Interpretation**:  One test ran and passed with no further explanation.
(Note how the additional properties are treated as if they were absent.)

## Formal spec

The following is a reasonably formal description of how exactly tapson
producers (programs that output it) and tapson consumers (programs that accept
it as input) should behave.  The examples above probably make the idea quite
clear, but just in case you're stuck on a detail and want the "word of God",
here are all the lawyerish specifics.

### Format

One JSON object per line.  The end of each line (including the last one) is
marked with a line feed character (`U+000A`, also known as a newline).  The end
of output is marked with an end of transmission character (`U+0004`, also known
as `^D` or EOF).

Compulsory properties (each object must contain one or both of these):

-   `ok`::`boolean` Whether the test passed.
-   `id`::`string` A unique identifier for the test, used for planned tests.

Optional properties (object may contain any, all or none of these):

-   `expected`::`string` What the test expected to find.
-   `actual`::`string` What the test found.

These properties being present with the wrong type should raise an error.
Other properties may be present, but must be assumed to be meaningless.

The contents of the `expected` and `actual` properties are intended for human
consumption.  Tapson consumers must not assume them to be in any particular
format.  Tapson producers must assume they will be read by a human.

The properties `id` and `ok` are OK for machine consumption.

As per the JSON specification, properties may appear in any order within
objects, and [Unicode][4] must be supported.

### Semantics

Any `id` property (if present) should be as universally unique as feasible to
the test that the object represents.  A long random string, for example.  [UUID
v4][5] is a good choice.  These must be consistent within the same stream, but
do not have to be consistent across multiple invocations of the tests.

The `id` property being present in an object *without* the `ok` property
represents a *planned test*, which results should be expected with a
corresponding `id` later in the same stream.  An object with *both* the `id`
and the `ok` property implies the test was previously planned with the same
`id` and its result is now available.  If the end of output is encountered
before some test plans have matching results, they must be treated as if they
failed with no further details given.  Any test results with `id`s that do not
correspond to any earlier plan must be treated as if they had no `id` property.
A test plan must not contain the properties `ok` or `actual`.  A result
matching a previously planned test must not contain the `expected` property,
but may otherwise.

Tapson streams contain no protocol version metadata.  Tapson producers and
consumers must clearly declare what versions they support.

Programs that operate by transforming tapson streams are *not required* to
preserve unspecified object properties present in their inputs, but may do so
if they wish.

The semantic meaning of tests' order is deliberately not specified.  How to
present tapson results for human consumption is deliberately not specified.

## FAQ

**Nested tests?**  Run separate test sets and merge the output streams.

**Timing information?**  Use the `actual` property.

**Marking tests as "TODO"?**  Don't include them.

**Comments?**  Don't include them.

**Bailing before tests finish**  Produce a failed test and emit an EOF.

## Organisation

Extension proposals, clarification requests, complaints, etc. should all be
discussed in the Github issue tracker.

Git commits in this repository tagged with a valid [semantic version][6] are
actual versions of the standard.

Semantic versioning 2.0.0 applies.  This means:

-   A major version bump implies the new standard is not backward-compatible
    with previous versions.  (For example, if a property is renamed.)
-   A minor version bump implies a backward-compatible improvement (For
    example, if a new JSON object field is introduced as an optional feature.)
-   A patch version bump implies wording or typographical tweaks, or addition
    of extra information, with no change in meaning for the standard itself.

If you write a tool that consumes or produces tapson, please make it clear to
users what tapson versions it works with.

[MIT license][7].

[1]: https://testanything.org/
[2]: http://www.json.org/
[3]: https://testanything.org/
[4]: http://unicode.org/
[5]: https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_.28random.29
[6]: http://semver.org/
[7]: http://opensource.org/licenses/MIT
