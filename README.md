> `leb128-nv` is developed and published from this repository.  It began in
> the novo-lang monorepo under `orbit/leb128-nv` and graduated out of it with
> its history; issues and pull requests belong here.

# leb128-nv

Unsigned LEB128 varints, over the full 64-bit range. LEB128 is how
DWARF and WebAssembly write every length, index and offset: a number is
cut into 7-bit groups, least significant first, and each byte's top bit
says whether another group follows. Everything below 128 is one byte,
which is what earns the encoding its arithmetic in a format full of
small numbers.

```novo
use leb128

fn main() [io]
    let wire = leb128.encode(624485)          // e5 8e 26
    match leb128.decode(wire)
        Ok(v)  => println("${v}")             // 624485
        Err(e) => println(e.message())
```

```
novo pkg add leb128-nv
```

## What it gives you

The API is on [the package's page](https://novo-lang.org/packages/leb128-nv),
generated from these sources: every `pub` declaration with its signature,
its effect row and the comment block written above it. A table of names
here would be a second original, and the second original is the one that
goes stale.

`Leb128Error` has two variants and both are input faults: `Truncated`
when the input ends with a continuation bit still set, `Overflow` when
the groups read exceed 64 bits. Each carries the byte count reached, so
a caller reporting a bad frame can say where.

## Writing into a buffer you already hold

`encode_into` and `decode_from` take a cursor and allocate nothing. The
offset lives in the cursor rather than in a third parameter because a
`Bytes` passed as a parameter is borrowed — the caller still holds it,
so a write through it lands in a copy the caller never sees. A cursor
owns its buffer, so a write through one lands where the caller can read
it. That is the same rule the standard library's own writers follow.

```novo
use leb128
use std.bytes

// Three lengths into one frame, back to back.
fn frame(a: Int, b: Int, c: Int) -> Bytes
    var w = bytes.cursor_le(bytes.zeros(3 * leb128.max_len()))
    let n = leb128.encode_into(w, a) + leb128.encode_into(w, b)
              + leb128.encode_into(w, c)
    bytes.slice(w.finish(), 0, n)
```

`decode_at` is the same question asked by offset rather than by
cursor — for a table of varint-encoded entries, or a format that hands
out each record's start. It answers a `Decoded` carrying `value` and
`len`, so the next offset is `off + d.len`:

```novo
fn walk(buf: Bytes) -> Result<Int, Leb128Error>
    var off = 0
    var total = 0
    while off < bytes.len(buf)
        let d = leb128.decode_at(buf, off)!
        total = total + d.value
        off = off + d.len
    Ok(total)
```

Reading with a cursor is the mirror image of writing, and the cursor's
position after each call is where the next number starts:

```novo
fn first_two(wire: Bytes) -> Result<Int, Leb128Error>
    var r = bytes.cursor_le(wire)
    let a = leb128.decode_from(r)!
    let b = leb128.decode_from(r)!
    Ok(a + b)
```

## The unsigned range, and negative `Int`s

`Int` is 64 bits and signed; LEB128 here is unsigned. A negative `Int`
is therefore read as the bit pattern of a value at or above 2^63, and
encodes to the ten bytes that value needs — `encode(-1)` is
`ff ff ff ff ff ff ff ff ff 01`, the encoding of 2^64 − 1. That is the
convention every LEB128 producer uses for u64, and it is what makes
`decode` a total inverse of `encode` over the whole range.

Signed LEB128, whose top group is sign-extended, is a different
encoding and is not here. Fold the sign away with `zigzag-nv` and
encode the result, which is what Protocol Buffers does.

## What it costs

One loop iteration per byte, a shift and a mask each way, plus one
bounds test per byte while decoding. `encode_into` and `decode_from`
allocate nothing; `encode` allocates its one result buffer. No table,
no state, and nothing that needs a heap — the arithmetic is `Int`
throughout.

`encode_into` panics when the cursor has less room than the value
needs, exactly as `c.put_u8` and `xs[i]` do: a buffer the caller sized
wrong is a mistake in the program. Reserve `max_len()`, or test
`dst.remaining()` first. Malformed **input** is never a panic — that is
what `Leb128Error` is for.

## What it does not do

No signed LEB128 and no fixed-width padding (the "pad to five bytes so
the value can be back-patched" trick some assemblers use). No streaming
across buffer boundaries: a varint must be contiguous, and a reader
that has only part of one gets `Truncated` and should wait for more
bytes rather than resume mid-number.

## Tests

```
novo test tests/leb128_tests.nv
```

The vectors are DWARF 4 §7.6's own table of unsigned encodings, copied
value for value, plus both ends of the 64-bit range and the two ways an
input can be malformed. Testing against the specification's numbers
rather than against this encoder's own output is what makes the suite
evidence that the bytes are LEB128 and not merely self-consistent.
