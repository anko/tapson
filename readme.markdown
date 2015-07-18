# tapson specification

A simple protocol for communicating the results of software tests.  Like
[TAP][1] but simpler, in [JSON][2], and supports streaming asynchronous tests'
results.

## In summary

`\n`-delimited JSON objects.

An object that contains an `ok` property is a *test result*.  One that doesn't
is a *test plan* and must contain an `id` that a later test result might match.

The `expected` property tells a human what the test expects to see.  The
`actual` property tells a human what actually happened.

Questions, gripes?  Create a github issue.

## Why

There was no equivalent existing protocol.

The closest is [TAP][3], but tapson is better in these ways:

-   Tapson is based on JSON.  (TAP uses a custom format.)
-   Tapson is better specified.  (The TAP specification is frequently vague,
    which has caused implementations to diverge.)
-   Tapson is designed with asynchronous operation in mind.  (TAP assumes
    planned tests are immediately executed.)
-   Tapson is deliberately simple.  (TAP considers YAML blocks, "TODO"
    annotations and test-skipping as parts of the protocol.)

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
clear, but for the lawyerish specifics, here you go.

### Format

One JSON object per line.  The end of each line (including the last one) is
marked with a line feed character (`U+000A`, also known as a newline).  The end
of output is marked with an end of transmission character (`U+0004`, also known
as `^D` or EOF).

Compulsory properties (each object must contain either or both of these):

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

### Expectations

Any `id` property (if present) should be as universally unique as feasible to
the test that the object represents (a long random string, for example).  These
must be consistent within the same stream, but do not have to be consistent
across multiple invocations of the tests.

The `id` property being present in an object *without* the `ok` property
represents a *planned test*, which results should be expected with a
corresponding `id` later in the same stream.  An object with *both* the `id`
and the `ok` property implies the test was previously planned with the same
`id` and its result is now available.  If the end of output is encountered
before some test plans have matching results, they must be treated as if they
failed with no further details given (only an `ok` property).  If any results
for non-existant planned tests are encountered, they must be treated as they
had no `id` property.)  A planned test must not contain the properties `ok` or
`actual`.  A result matching a previously planned test must not contain the
`expected` property.

A tapson stream contains no protocol version metadata.  Tapson producers and
consumers must declare what versions they support.

Programs that operate by transforming tapson streams are *not required* to
output unspecified properties present in the objects, but may do so if they
wish.

The semantic meaning of tests' order is not specified.  The way to present
tapson results for human consumption is left up to implementors.

## FAQ

**Nested tests?**  Run separate test sets and merge the output streams.

**Timing information?**  Use the `actual` property.

**Marking tests as "TODO"?**  Don't include them.

**Comments?**  Don't include them.

**Bailing before tests finish**  Produce a failed test and emit an EOF.

## Organisation

Extension proposals, clarification requests, complaints, etc. should all be
discussed in the Github issue tracker.

Git commits in this repository tagged with a valid [semantic version][5] are
actual versions of the standard.  Semantic versioning 2.0.0 applies.

If you write a tool that consumes or produces tapson, please make it clear to
users what version it complies to.

[MIT license][6].

[1]: https://testanything.org/
[2]: http://www.json.org/
[3]: https://testanything.org/
[4]: http://unicode.org/
[5]: http://semver.org/
[6]: http://opensource.org/licenses/MIT
