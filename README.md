# leb128-nv

LEB128 is a variable-length encoding of an integer. A number is cut
into groups of seven bits, and each group travels in one byte. It is
what DWARF and the WebAssembly binary format write for every length,
index and offset. The encoding is specified in the
[DWARF standard](https://dwarfstd.org/) section 7.6, which also
publishes the table of examples this package is tested against. This
package brings the unsigned form, over the full 64-bit range, to
novo-lang.

## What LEB128 is

A number is cut into groups of seven bits, least significant group
first. Each group travels in the low seven bits of one byte. The top
bit of that byte is the **continuation bit**. It is 1 on every byte but
the last, so a reader finds the end of a number without being told its
length beforehand.

Small numbers are short. Everything below 128 fits in one group and so
takes one byte. The length steps up at every seventh power of two and
nowhere else. That is what earns the encoding its arithmetic in a
format full of small numbers.

There is an unsigned form and a signed form. The unsigned form is the
one here. The signed form sign-extends its top group instead, so the
two are different encodings of the same number and a reader has to know
which it is holding.

An encoding is not required to be the shortest one. A group of zeroes
may be added on the end, with the continuation bit set on the group
before it, and the value is unchanged. Two byte strings can therefore
carry the same number.

| Quantity | Value |
| --- | --- |
| Bits one byte carries | 7 |
| Bytes a value below 128 takes | 1 |
| Bytes an unsigned 64-bit value takes, at most | 10 |
| Groups that fit in 63 bits | 9 |
| The length steps at | every seventh power of two |
| `624485` encoded | `e5 8e 26` |
| `2^64 - 1` encoded | `ff ff ff ff ff ff ff ff ff 01` |

## Install

```
novo pkg add leb128-nv
```

## Example

```novo
use leb128

fn main() [io]
    // Encode a number into a fresh buffer of exactly its own length.
    let wire = leb128.encode(624485)          // e5 8e 26
    // Read it back. A malformed input arrives as an `Err`.
    match leb128.decode(wire)
        Ok(v)  => println("${v}")             // 624485
        Err(e) => println(e.message())
```

Build and test with `novo pkg build` and `novo test tests/leb128_tests.nv`.

## What the package contains

| Module | Contents |
| --- | --- |
| `leb128` | The two size calls, `max_len` and `encoded_len`. The two encoders, `encode` and `encode_into`. The three decoders, `decode`, `decode_from` and `decode_at`. The `Decoded` pair and the `Leb128Error` enum. |

The API reference is on
[the package's page](https://novo-lang.org/packages/leb128-nv). `novo
doc` generates it from these sources: every `pub` declaration with its
signature, its effect row and the comment block written above it. Each
entry carries a worked example that is compiled and run as a test.

## How to choose an entry point

**`encode` and `decode` take the whole value at once.** `encode`
allocates a buffer of exactly the length the value needs and answers
it. `decode` reads the varint at the start of a buffer and ignores
whatever follows. This is the pair for a program that is building or
reading one message.

**`encode_into` and `decode_from` work over a cursor the caller
owns.** Each allocates nothing, and each leaves the cursor on the byte
after the number. This is the pair for a frame writer, and for a reader
walking a run of varints with one cursor.

**`decode_at` asks by offset instead of by cursor.** It answers a
`Decoded`, which carries the value and the number of bytes it occupied,
so the next offset is `off + d.len`. This is the call for a table of
varint-encoded entries, or a format that hands out each record's start.

**`max_len` and `encoded_len` are the sizes.** `encoded_len` answers
what one value will take. `max_len` answers what any value can take,
which is the room to reserve for a value that is not yet computed.

## The rules a user needs

1. **A negative `Int` is read as an unsigned 64-bit value.** `Int` is
   64 bits and signed, and the encoding here is unsigned. `encode(0 -
   1)` is the ten bytes of 2^64 − 1. That is the convention every
   LEB128 producer uses for a 64-bit unsigned number, and it is what
   makes `decode` invert `encode` over the whole range.
2. **A malformed input is a value and never a panic.** `Truncated`
   says the input ended with a continuation bit still set. `Overflow`
   says the groups read exceed 64 bits. Each carries the byte count
   the reader reached, so a caller reporting a bad frame can say
   where.
3. **A cursor with less room than the value needs is a panic.**
   `encode_into` writes through `put_u8`, which panics on a short
   buffer exactly as `xs[i]` does on an index out of range. A buffer
   the caller sized wrong is a mistake in the program rather than
   input. Reserve `max_len()`, or test `dst.remaining()` first.
4. **The cursor's position after a call is where the next number
   starts.** A caller reading a run of varints needs nothing else. It
   calls `decode_from` again.
5. **An offset outside the buffer is `Truncated`, not a panic.** An
   offset that came out of the data being decoded is input, and input
   is never a reason to stop the program. `decode_at` answers
   `Truncated(after: 0)` for it.
6. **A padded encoding is accepted.** The ten bytes `80 80 80 80 80 80
   80 80 80 00` decode to 0, as the single byte `00` does. A program
   that needs one spelling per value compares the length it read
   against `encoded_len` of the value.
7. **A varint must be contiguous in one buffer.** A reader holding
   only part of one gets `Truncated`. It should wait for more bytes
   and read the number again from its first byte.
8. **Only `encode` allocates.** It allocates its one result buffer.
   The arithmetic is `Int` throughout, so nothing else here puts
   anything on the heap. Every function is arithmetic over bytes the
   caller already holds, which is why the layer is `core` and no
   function declares an effect.

## What is not included

- **Signed LEB128.** Its top group is sign-extended, which makes it a
  different encoding. Fold the sign away with
  [zigzag-nv](https://novo-lang.org/packages/zigzag-nv) and encode the
  result, which is what Protocol Buffers does.
- **A call that writes a padded encoding.** Some assemblers write a
  value in a fixed five bytes so that it can be back-patched later.
  `encode_into` writes the bytes the value needs and no more.
- **A refusal for a padded encoding.** A decoder that refused one
  would reject documents a conforming producer writes. A caller for
  whom the spelling matters checks the length itself.
- **Streaming across buffer boundaries.** A reader that has part of a
  varint cannot carry the partial state to the next buffer. It gets
  `Truncated` and reads the number again once the bytes are contiguous.
- **Any input or output.** Every function here reads and writes memory
  the caller already holds.

## Related packages

- [varint-nv](https://novo-lang.org/packages/varint-nv) is the ZigZag
  fold and this byte loop as one call, at 32 and 64 bits. It is what
  Protocol Buffers calls `sint32` and `sint64`, and it is what to reach
  for when the numbers are signed.
- [zigzag-nv](https://novo-lang.org/packages/zigzag-nv) is the fold on
  its own. It maps a signed integer onto an unsigned one so that a
  small negative number stays short.

## Tests

```
novo test tests/leb128_tests.nv
```

The suite is 14 tests. The vectors in it are the DWARF standard's own
table of unsigned LEB128 encodings, section 7.6, copied value for
value. Testing against the specification's numbers rather than against
this encoder's own output is what makes the suite evidence that the
bytes are LEB128 and not merely self-consistent.

Beside the vectors the suite asserts both ends of the 64-bit range, the
lengths at the group boundaries, that `encoded_len` agrees with
`encode` at all 64 powers of two, that `decode_at` reports the length
it read, and the three ways an input can be malformed. Every example in
a documentation comment is compiled by `novo doc` and run by `novo
test`, so an example that stopped being true is a failing test.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
