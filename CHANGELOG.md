# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.7 — 2026-09-25

`decode` and `decode_at` read the caller's buffer byte by byte where it
is, instead of through a cursor.  A cursor may write into the buffer it
holds, and the next Novo release refuses to build a writable cursor over
a buffer the caller passed in only to be read.  No signature changed,
nothing is copied, and every answer is what 0.1.6 gave.

## 0.1.6 — 2026-09-18

The documentation and comments in plain prose; no signature changed.

## 0.1.5 — 2026-09-08

- **The manifest names the layer.**  `layer = "core"` says that the
  public API requires no effects, and `novo pkg publish` checks the code
  against that.  No code changed.  The layers are described under Design
  in the [publishing guide](https://novo-lang.org/docs/publishing.html#design).
- **The sources in the canonical form** `novo fmt` prints today.  The
  change is spacing and alignment only, and no code changed.

## 0.1.4

The API reference generated from the code, with the examples in it run
as tests.  No code changed, and every signature and every byte on the
wire is what 0.1.3 shipped.

- **Every `pub` item documented under Go's rule**, in the comment block
  directly above the declaration, its first sentence the summary a
  reader meets before opening anything.  `novo doc` turns that into
  [the package's page](https://novo-lang.org/packages/leb128-nv).
- **Seven worked examples, and they run.**  Both pairs are shown where
  they are declared.  `encode` and `decode` allocate, `encode_into` and
  `decode_from` work over a cursor, and `decode_at` asks by offset.  A
  truncated input is shown answering rather than crashing.  A fenced
  `novo` block in a documentation comment is compiled by `novo doc` and
  run by `novo test src/leb128.nv`, so an example that stopped being
  true is a failing test.

## 0.1.3

The Apache-2.0 text in the tarball, and the sources released from
`novolang/leb128-nv`.  No code changed, and every signature and every
byte on the wire is what 0.1.2 shipped.

- **`LICENSE` ships with the package.**  It is on the publish
  allow-list, so the tarball carries the Apache-2.0 text rather than
  only naming it in the manifest.
- **The sources, CI and the release tags live in
  `novolang/leb128-nv`** from this version on.

## 0.1.2

The bit arithmetic written with the operators, and the test suite out of
`src/`.  The bytes are unchanged and every signature is the same.  The
DWARF vectors, both ends of the 64-bit range and the two malformed
inputs all still pass, which is what says so.

- **The bit arithmetic is written with the operators.**  `bits.band`,
  `bits.bor`, `bits.shl` and `bits.shr` become `&`, `|`, `<<` and `>>>`,
  and the two accumulator loops use `>>>=` and `|=`.  The encoder's
  group extraction reads `rest & 0x7f`, which is the line the DWARF
  standard writes.  Nothing about the encoding changed.
- **The test module moved out of `src/`.**  A package's `src/` ships
  whole and a consumer compiles every module in it, so the suite is
  under `tests/` where it is not published.  Run it with
  `novo test tests/leb128_tests.nv`.

## 0.1.1

`decode_at`, which reads a varint at an offset.  Everything `0.1.0`
shipped keeps its name, its signature and its result, and nothing here
needs a toolchain `0.1.0` did not.  A consumer written against `^0.1`
picks this up on its next `novo pkg update` and has nothing to change.

- **`decode_at(src, off)` answers the value and its length**, as a
  `Decoded`.  `decode_from` already leaves the length in the cursor's
  position, and a caller walking a buffer by offset had no way to ask.
  A table of varint-encoded entries and a format that hands out each
  record's start are two such callers.  An offset outside the buffer is
  `Truncated` rather than a panic, because an offset that came out of
  the data being decoded is input.

## 0.1.0

The unsigned encoding: `max_len`, `encoded_len`, `encode_into`,
`encode`, `decode_from`, `decode`, and the `Leb128Error` pair.

- **The full unsigned 64-bit range**, with a negative `Int` read as the
  bit pattern of a value at or above 2^63.
- **The in-place pair allocates nothing.**  `encode_into` and
  `decode_from` work over a cursor the caller owns.
- **A malformed input is a value.**  `Truncated` and `Overflow` each
  carry the byte count reached, and nothing about bad input panics.
- **The vectors are the DWARF standard section 7.6's**, so the suite is
  evidence about the format rather than about this implementation.
