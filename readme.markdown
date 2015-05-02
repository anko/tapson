# tapson

Simple protocol for communicating the results of software tests.  Like [TAP][1]
but simpler and in [JSON][2].

## Spec

### Format

One JSON object per line.  The end of each line (including the last one) is
marked with a line feed character (`U+000A`, also known as a newline).  The end
of output is marked with an end of transmission character (`U+0004`, also known
as `^D` or EOF).

Compulsory fields in each object:

-   `ok`::`boolean` Whether the test passed.

Optional fields:

-   `description`::`string` What was tested.
-   `expected`::`string` What the test expected to find.
-   `actual`::`string` What the test found.

All other fields are disallowed.

The contents of the `description`, `expected` and `actual` fields are intended
for human consumption.  Tapson consumers must not assume them to be in any
particular format.  Tapson producers must assume these fields will be read by a
human.

The JSON objects are not guaranteed to have the same optional fields— they may
be omitted or included on an object-by-object basis.

As per the JSON specification, fields may appear in any order within objects
and [Unicode][3] must be supported.

A test being present in a tapson stream must imply it succeeded or failed. A
test being absent from a tapson stream must not imply success or failure.

### Versioning

A tapson stream has no version header.  Tapson producers and consumers must
declare what versions they support.

## FAQ

**Nested tests?**  Use the description field.

**Timing information?**  Use the `actual` field.

**Narking tests as "TODO"**  Don't include them.

**Bailing before finishing all tests**  Simply emit an EOF

## Rationale

[TAP 13][5] is poorly specified and complicated by unnecessary features.  The
solution is clearly [another standard][6] and to piggyback on existing tech.

Advantages of tapson over TAP:

-   Almost anything has a JSON parser.  What doesn't should.
-   Newline-separated tests without header or metadata noise are easy to parse,
    concatenate, slice into parts, and filter.

Disadvantages of tapson over TAP:

-   No indication of what tests are planned.

[1]: https://testanything.org/
[2]: http://www.json.org/
[3]: http://unicode.org/
[4]: http://semver.org/
[5]: https://testanything.org/tap-version-13-specification.html
[6]: https://xkcd.com/927/
