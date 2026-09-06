# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.4

Documentation: the reference is generated from the code, and the
examples in it are doctests.  No code changed — every signature, and
every byte on the wire, is what 0.1.3 shipped.

- **Every `pub` item is documented under Go's rule**, the comment block
  directly above the declaration, its first sentence the summary a
  reader meets before opening anything.  `novo doc` turns that into
  [the package's page](https://novo-lang.org/packages/leb128-nv).
- **Seven worked examples, and they run.**  Both pairs are shown where
  they are declared — `encode` / `decode` allocating, `encode_into` /
  `decode_from` over a cursor, `decode_at` by offset — along with what
  a truncated input answers.  A fenced `novo` block in a documentation
  comment is compiled by `novo doc` and run by
  `novo test src/leb128.nv`, so an example that stopped being true is a
  failing test rather than a reader's afternoon.

## 0.1.3

Developed in its own repository from this version.  `novolang/leb128-nv` is
where the sources live, where CI runs and where releases are tagged,
and the novo-lang monorepo no longer carries a copy.  No code changed:
every signature, and every byte on the wire, is what 0.1.2 shipped.

- **`LICENSE` ships with the package.**  It is on the publish
  allow-list, so the tarball now carries the Apache-2.0 text rather
  than only naming it in the manifest.

## 0.1.2

A patch: the bytes are unchanged and every signature is the same.  The
DWARF vectors, both ends of the 64-bit range and the two malformed
inputs all still pass, which is what says so.

- **The bit arithmetic is written with the operators.**  `bits.band`,
  `bits.bor`, `bits.shl` and `bits.shr` become `&`, `|`, `<<` and
  `>>>`, and the two accumulator loops use `>>>=` and `|=`.  The
  encoder's group extraction now reads `rest & 0x7f`, which is the line
  the DWARF specification writes.  Nothing about the encoding changed.
- **The test module moved out of `src/`.**  A package's `src/` ships
  whole and a consumer compiles every module in it, so the suite is
  under `tests/` where it is not published.  Run it with
  `novo test tests/leb128_tests.nv`.

## 0.1.1

A patch, deliberately.  Everything `0.1.0` shipped keeps its name, its
signature and its result, and nothing here needs a toolchain `0.1.0` did
not — so a consumer written against `^0.1` picks this up on its next
`novo pkg update` and has nothing to change.

- **`decode_at(src, off)` answers the value AND its length**, as a
  `Decoded`.  `decode_from` already leaves the length in the cursor's
  position, but a caller walking a buffer by offset — a table of
  varint-encoded entries, a format that hands out each record's start —
  had no way to ask.  An offset outside the buffer is `Truncated`
  rather than a panic, because an offset that came out of the data
  being decoded is input.

## 0.1.0

First release: `max_len`, `encoded_len`, `encode_into`, `encode`,
`decode_from`, `decode`, and the `Leb128Error` pair.

- **The full unsigned 64-bit range**, with a negative `Int` read as the
  bit pattern of a value at or above 2^63.
- **The in-place pair allocates nothing.**  `encode_into` and
  `decode_from` work over a cursor the caller owns.
- **A malformed input is a value.**  `Truncated` and `Overflow` each
  carry the byte count reached; nothing about bad input panics.
- **The vectors are DWARF 4 §7.6's**, so the suite is evidence about
  the format rather than about this implementation.
