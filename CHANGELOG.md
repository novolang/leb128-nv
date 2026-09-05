# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

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
