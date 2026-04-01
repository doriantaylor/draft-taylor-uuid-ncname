---
title: Compact UUIDs for Constrained Grammars
abbrev: uuid-ncname
cat: info

docname: draft-taylor-uuid-ncname-05
submissiontype: IETF
date: 2026
consensus: true
v: 3
keyword:
  - uuid
  - ncname
  - base32
  - base64
  - base58
pi: [toc, sortrefs, symrefs]
workgroup: Independent

author:
  - name: Dorian Taylor
    email: ietf@doriantaylor.com
    uri: https://doriantaylor.com/
  - name: Kyzer R. Davis
    email: kydavis@cisco.com
    org: Cisco Systems

normative:
  RFC9562:
  RFC4648:
  Base58:
    target: https://github.com/bitcoin/bitcoin/blob/master/src/base58.cpp
    title: Base58 Bitcoin
    author:
    - org: Bitcoin
    date: 2008-11
    seriesinfo:
      commit: fae71d3

informative:
  XML-NAMES:
    target: https://www.w3.org/TR/2009/REC-xml-names-20091208/
    title: Namespaces in XML 1.0 (Third Edition)
    author:
    - ins: T. Bray
      name: Tim Bray
    - ins: D. Hollander
      name: Dave Hollander
    - ins: A. Layman
      name: Andrew Layman
    - ins: R. Tobin
      name: Richard Tobin
    - ins: H S. Thompson
      name: Henry S. Thompson
    date: '2009-12-08'

--- abstract

This document specifies an alternative compact representation for Universally Unique Identifier (UUID) from RFC9562 that preserves some properties of the UUID canonical format to fit more restrictive grammar contexts such as the non-colonized name (NCName) grammar production, which is pervasive in eXtensible Markup Language (XML).

--- middle

# Introduction {#introduction}

The formal grammar production "one or more letters or underscores followed by zero or more letters, digits, or underscores" (denoted by the regular expression `/^[A-Za-z_][0-9A-Za-z_]*$/`) is ubiquitous in computing.
It is often used for identifiers, and for good reasons.
One may encounter some variations on this theme, like admitting hyphens, dots, or Unicode alphanumerics.
Some systems may impose additional constraints, like case-sensitivity (or the lack of it), explicit upper- or lower-case letters, or limits on identifier length.

UUIDs are standardized 128-bit identifiers with many useful properties, and there are many places where it would make sense to use them, but their canonical representation, either with or without [the URN prefix](#RFC9562), does not conform to the grammar constraints described above:

{: spacing="compact"}

- UUIDs contain hyphens (and colons, in UUID URNs),
- UUIDs may potentially start with a digit,
- UUIDs may potentially be too long for a given slot.

This leads to developers creating ad-hoc, overlapping, and generally mutually incompatible solutions.

The goal of this specification is to address an ostensible need for a UUID representation which:

{: spacing="compact"}
- Produces a format based on a given UUID algorithm that can be encoded/decoded to and from the canonical UUID representation, and is thus isomorphic to it.
- Produces an output that is fewer characters in length than the canonical form (even after removing the hyphens), and thus more compact.
- Ensures that the output format always starts (and ends) with a letter (`/^[A-Za-z]/`) and contains only letters and digits (`/[A-Za-z0-9]/`), so the value can be used in contexts where the aforementioned grammar is required.
- Provides different encoding variants to accommodate different use cases, such as case-sensitive and case-insensitive.
- Is amenable to detection and identification by heuristics, in a manner analogous to the canonical UUID representation.

This document specifies a strategy for a compact representation of UUIDs, with multiple encoding variants, as well as the related transformations to and from the familiar UUID format.
These alternative UUID formats are suitable methods for uniquely identifying entities in a symbol space large enough that the identifiers do not collide.

The proposed name for the general strategy is "UUID-NCName", after the [NCName grammar production](#XML-NAMES), which is pervasive in XML and
RDF applications.
The encodings are thus styled as "UUID-NCName-32" for {{RFC9562, Section 6}}, "UUID-NCName-58" for {{Base58}}, and "UUID-NCName-64" for {{RFC4648, Section 5}}, referring to the numerical base of their respective encodings.

## Motivation & Applications {#applications}

The purpose of a large, generated identifier like the UUID is to satisfy the uniqueness criterion while also specifying a datatype and normal form for said identifiers, and ultimately alleviate the need to set time aside to think up names for things.

Why one would want to go inserting UUIDs in places they would not otherwise fit, is so these UUIDs can be cross-referenced in some other database where they *do* fit.

Consider:

{: spacing="compact"}
- A programming environment that separates the task of writing logic from naming things, stores identifiers internally as UUID-NCName-32 prior to transforming them on display or export, thus preserving the correctness of the syntax.
- A component content management system that uses UUIDs to identify elementary content components, uses the UUID-NCName-64 (or UUID-NCName-58) representations of the same UUIDs as fragment identifiers for when those components are transcluded.

# Terminology {#terminology}

{::boilerplate bcp14-tagged}

# Encoding Strategy {#strategy}

Not all 128 bits of a UUID are data; rather, several bits are masked.

{: spacing="compact"}
- Up to three high bits in the following segment, called `var`, specify the variant of the UUID, which indicates the layout of the UUID and thus how it should be read.
- Within the IETF and DCE variant, the top four bits of the third segment, known as `ver`, specify the UUID's version, which are fixed values that indicate the algorithm used to generate the UUID, and thus how it should be read.

UUID-NCName removes these two masked quartets and uses them as "bookends" {{bookends}} for the rest of the identifier, mapping them to the first sixteen symbols of the Base32 table {{RFC4648}}, which are all letters.

For the variant, round up to four bits, so that the bookend characters are always the same for a given UUID, regardless of the number of variant bits set.
These "bookend" characters provide an analogous hint to a developer about the nature of the UUID, just as one can by inspecting the third and fourth segments of a canonical hexadecimal UUID representation.

Next, bit-shift the remaining 120 bits to close the gaps of the two masked quartets previously removed.

The resulting 120 bits divide evenly by both 5 (for Base32, yielding 24 symbols) and 6 (for Base64, yielding 20 symbols).

For Base58, note that the encoding cannot map to an even number of bits, but Base58 does not have the same concerns with regard to padding as Base32 and Base64.
Indeed, in Base58 there is a different padding issue: some inputs yield shorter outputs than others.
To address this, pad the Base58 representation by appending with underscore characters (`_`, a character *not* in the Base58 alphabet) to get a consistent length.
The details are laid out in the [encoding algorithm](#encoding) section.

Given the example UUID from {{RFC9562, Section 4}}: `f81d4fae-7dec-11d0-a765-00a0c91e6bf6` the transformation yields the following results shown in {{exampleTable}}.

| Value          | Encoding                   | Properties                      |
|----------------|----------------------------|---------------------------------|
| UUID-NCName-32 | b7aou7lt55qoqoziauder427wk | 26 Characters, case-insensitive |
| UUID-NCName-58 | B7wc88dU4e3NyJEj3e944DK    | 23 Characters, case-sensitive   |
| UUID-NCName-64 | B-B1Prn3sHQdlAKDJHmv2K     | 22 Characters, case-sensitive   |
{: #exampleTable title='Example of the transformation of a UUID to UUID-NCName representations'}

For more examples see the test vectors appendix: {{testVectors}}.

## Syntax {#syntax}

Below is the ABNF grammar for the productions `uuid-ncname-32`, `uuid-ncname-58`, and `uuid-ncname-64`:

~~~~ abnf
uuid-ncname-32 = bookend 24base32 bookend
uuid-ncname-58 = bookend base58 bookend
uuid-ncname-64 = bookend 20base64url bookend
bookend        = %x41-50 / %x61-70 ; [A-Pa-p]
base32         = %x32-37 / %x41-5a / %x61-7a ; [2-7A-Za-z]
b58char        = %x31-39 / %x41-48 / %x4a-4e / %x50-5a /
                 %x61-6c / %x6d-7a ; [1-9A-HJ-NP-Za-km-z]
base58         = 15b58char 6"_" / 16b58char 5"_" /
                 17b58char 4"_" / 18b58char 3"_" /
                 19b58char 2"_" / 20b58char "_" / 21b58char
                 ; (symbol sequence plus appropriate padding)
base64url      = %x2d / %x30-39 / %x41-5a / %x5f / %x61-7a
                 ; [-0-9A-Z_a-z]
~~~~

## Bookends {#bookends}

"Bookends" are 4-bit sequences (nybbles, quartets, etc.) which are mapped directly onto the first sixteen symbols of the Base32 table from {{RFC4648, Section 6}} [A-P].

This portion of the Base32 table is identical to that found in the Base64 Table {{RFC4648, Section 4}}, though the term Base32 is used to underscore the fact that bookend characters are case-insensitive.
This is because certain environments encode meaning into the case of the first character of a symbol, so it is important that its literal representation be flexible.

However, there is little value in arbitrarily constraining the last character.

Nevertheless, UUID-NCName-32 symbols SHOULD be generated entirely lower-case, while UUID-NCName-58 and UUID-NCName-64 symbols SHOULD be generated with the bookend characters in upper-case.

Random UUIDs, UUIDv4, for instance, will always begin with `e/E`.
For other versions from {{RFC9562, Section 4.2}} refer to {{bookendTable}}.

| IETF/DCE Version           | Bookend Character   | Description                     |
|----------------------------|---------------------|---------------------------------|
| `0,    0x00`               | a/A                 | Nil                             |
| `1,    0x01`               | b/B                 | Gregorian Timestamp             |
| `2,    0x02`               | c/C                 | DCE "Security"                  |
| `3,    0x03`               | d/D                 | Name-Based MD5                  |
| `4,    0x04`               | e/E                 | Random                          |
| `5,    0x05`               | f/F                 | Name-Based SHA-1                |
| `6,    0x06`               | g/G                 | Reordered Gregorian Timestamp   |
| `7,    0x07`               | h/H                 | Unix Timestamp                  |
| `8,    0x08`               | i/I                 | Custom                          |
| `9-15, 0x09-0x0F`          | j-p/J-P             | Reserved for Future definitions |
{: #bookendTable title='Bookend Characters for UUID Versions'}

Any UUID with the variant bits set as defined in {{RFC9562, Section 4.1}}, row 2 will always terminate with one of `i/I`, `j/J`, `k/K`, or `l/L`.
Review {{variantTable}} for all possible variant bit combinations.

| Variant Bits              | Bookend Character     | Description           |
|---------------------------|-----------------------|-----------------------|
| 0-7,     0b0000 - 0b0111  | a-h/A-H               | NCS and NIL UUID      |
| 8-9,A-B, 0b1000 - 0b1011  | i-l/I-L               | IETF, DCE, and Others |
| C-D,     0b1100 - 0b1101  | m-n/M-N               | Microsoft             |
| E-F,     0b1110 - 0b1111  | o-p/O-P               | Reserved and MAX UUID |
{: #variantTable title='Bookend Characters for UUID Variants'}

An example version/variant bookend layout for UUIDv4 follows the table where "M" represents the version placement for the BaseXX representation of 0x4 (0b0100) and "N" represents the variant placement for one of the four possible hexadecimal representations of variant 10xx: 0x8 (0b1000), 0x9 (0b1001), 0xA (0b1010), 0xB (0b1011) as its Base32, Base64, or Base58 symbol (`i/I`, `j/J`, `k/K`, or `l/L`)

~~~
// UUID-NCName-32
e000000000000000000000000i
e000000000000000000000000l
e000000000000000000000000j
e000000000000000000000000k
MxxxxxxxxxxxxxxxxxxxxxxxxN

// UUID-NCName-58
E000000000000000000000I
E000000000000000000000J
E000000000000000000000K
E000000000000000000000L
MxxxxxxxxxxxxxxxxxxxxxN

// UUID-NCName-64
E00000000000000000000I
E00000000000000000000J
E00000000000000000000K
E00000000000000000000L
MxxxxxxxxxxxxxxxxxxxxN
~~~
{: title='UUID-NCName Bookend Layout for UUIDv4'}

Another example can be seen in {{nilTestVectors}} for the nil UUID, which has all bits set to zero, and thus has the bookend characters `a/A` for both version and variant and for {{maxTestVectors}} for the MAX UUID, which has all bits set to one, and thus has the bookend characters `p/P` for both version and variant.

## Detection Heuristic {#detection}

It is possible to heuristically determine whether a given string is a UUID-NCName symbol, and if so, which encoding it uses.

All encodings of UUID-NCName are a fixed length:

{: spacing="compact"}
- UUID-NCName-32 is always 26 characters.
- UUID-NCName-58 is always 23 characters.
- UUID-NCName-64 is always 22 characters.

All encodings employ the same "bookend" mechanism.

The first and last character in all three representations will therefore always be the same, modulo case, for a given UUID.

Furthermore, since these "bookend" characters represent the version and variant bits, they will correspond to predictable values.

Given these facts, any UUID-NCName representation MAY be captured (and its "bookends" separated) using the following regular expression:

~~~perl
/\b([A-Pa-p]) # zero-width boundary and version bookend
([2-7A-Za-z]{24}|[-0-9A-Z_a-z]{20}| # base32 and base64
  (?:[1-9A-HJ-NP-Za-km-z]{15}_{6}|[1-9A-HJ-NP-Za-km-z]{16}_{5}|
     [1-9A-HJ-NP-Za-km-z]{17}_{4}|[1-9A-HJ-NP-Za-km-z]{18}___|
     [1-9A-HJ-NP-Za-km-z]{19}__|[1-9A-HJ-NP-Za-km-z]{20}_|
     [1-9A-HJ-NP-Za-km-z]{21})) # base58 with underscore pad
([A-Pa-p])\b/x # variant bookend and zero-width boundary
~~~

This detection method is considered a heuristic because it is possible to identify false-positive matches in random strings of text, just as
it would be with a canonical UUID representation.

It is assumed that there would be sufficient enough context to positively identify these alternative UUID representations in the wild.

## Equivalency {#equivalency}

Two UUID-NCName symbols are necessarily identical if they convert to the same (canonical) UUID.

Two UUID-NCName-32 symbols are identical if their string values match when normalized to either all upper- or lower-case letters.

Two UUID-NCName-58 or UUID-NCName-64 symbols are identical if their string values match when all characters, including the "bookend" characters, are normalized to either upper- or lower-case.

# Encoding Algorithm {#encoding}

These are algorithms for encoding and decoding the symbols, transforming them to and from the canonical UUID representation.

Equivalent algorithms no doubt exist, but these are the ones used in the [reference implementations](#implementations).


## Bit Shifting Algorithm {#bitShifting}
First apply the shifting algorithm:

1. Convert the UUID to a binary string `bin`.

2. Convert `bin` to an array of four 32-bit unsigned network-endian integers `ints`.

3. Extract `version` as `(ints[1] & 0x0000f000) >> 12`.

4. Extract `variant` as `(ints[2] & 0xf0000000) >> 24`.

5. Assign `ints[1] = (ints[1] & 0xffff0000) | ((ints[1] & 0x00000fff) << 4) | ((ints[2] & 0x0fffffff) >> 24)`.

6. Assign `ints[2] = (ints[2] & 0x00ffffff) << 8 | (ints[3] >> 24)`.

7. Assign `ints[3] = (ints[3] << 8) | variant`.

8. Convert `ints` back into a binary string and return it along with the `version`.

Pseudocode for these steps using the Generic UUID from {{RFC9562, Section 4}} as an example can be found in {{bitShiftExample}}.

## UUID-NCName-32 (Base32)

1. Take the binary string `bin` and shift the last octet to the right by one bit.

2. Encode `bin` with the Base32 algorithm to get the string `b32`.

3. Truncate `b32` to 25 characters, removing any padding.

4. Convert `version` to its value in the Base32 table.

5. Return `version` concatenated to `b32`, optionally in either upper or lower case.

Pseudocode for these steps using the Generic UUID from {{RFC9562, Section 4}} as an example can be found in {{uuidNcname32Example}}.

## UUID-NCName-58 (Base58)

1. Remove the last octet from the binary string `bin`, convert it to an integer and assign it to `variant`.

2. Shift `variant` to the right by 4 bits, and convert it to its value in the Base32 table.

3. Encode the remaining `bin` with the Base58 algorithm to get the string `b58`.

4. If `b58` is less than 21 characters long, append underscores (`_`) until it is.

5. Convert `version` to its value in the Base32 table.

6. Return the concatenation of `version`, `b58`, and `variant`.

Pseudocode for these steps using the Generic UUID from {{RFC9562, Section 4}} as an example can be found in {{uuidNcname58Example}}.

## UUID-NCName-64 (Base64url)

1. Take the binary string `bin` and shift the last octet to the right by two bits.

2. Encode `bin` with the base64url algorithm to get the string `b64`.

3. Truncate `b64` to 21 characters, removing any padding.

4. Convert `version` to its value in the Base32 table.

5. Return `version` concatenated to `b64`.

Pseudocode for these steps using the Generic UUID from {{RFC9562, Section 4}} as an example can be found in {{uuidNcname64Example}}.

# Decoding Algorithm {#decoding}

1. First use the {{detection}}{: format="title"} to determine whether the symbol `ncname` is Base32, Base58, or Base64.

2. Remove the first character of the symbol `ncname` and convert it into an integer according to the Base32 spec; call that integer `version`.

3. If `ncname` is Base58:

   {:type="a"}
   1. Remove the last character and decode it to an integer according to the Base32 spec; call that integer `variant`.

   2. Shift `variant` four bits to the left.

   3. Remove all trailing underscores from the remainder of `ncname`.

   4. Decode the remainder of `ncname` with the Base58 algorithm as `bin`.

   5. Append the octet corresponding to the value of `variant` to `bin`.

4. Otherwise:

   {:type="a"}
   1. If `ncname` is Base64, and the last character is lowercase, set it to uppercase.

   2. Append padding if necessary to satisfy the decoder, `A======` for Base32 and `A==` for Base64.

   3. Decode the remainder of `ncname` by either the `base32` or `base64url` decoding algorithm into binary string `bin`.

   4. If `ncname` is Base32, shift the last octet of `bin` one bit to the left; if Base64 shift it two bits.

Then, apply the shifting algorithm in reverse:

{:start="5"}
5. Ensure `version &= 0xf` so it is in the range of 0-15.

6. Convert the binary string `bin` into an array of four 32-bit unsigned network-endian integers `ints`.

7. Assign `variant = (ints[3] & 0xf0) << 24`.

8. Shift and assign `ints[3] >>= 8`.

9. Union and assign `ints[3] |= ((ints[2] & 0xff) << 24)`.

10. Shift and assign `ints[2] >>= 8`.

11. Union and assign `ints[2] |= ((ints[1] & 0xf) << 24) | variant`.

12. Assign `ints[1] = (ints[1] & 0xffff0000) | (version << 12) | ((ints[1] >> 4) & 0xfff)`.

13. Convert `ints` back into the new binary string `bin`.

14. Format `bin` as a canonical UUID.

# IANA Considerations {#iana}

There are no discernible IANA considerations associated with this specification.

# Security Considerations {#security}

As UUID-NCName symbols are isomorphic to their canonical UUID representations, the security considerations for these symbols are also the same as {{RFC9562}}.

--- back

## Pseudocode

Given the example UUID from {{RFC9562, Section 4}}: `f81d4fae-7dec-11d0-a765-00a0c91e6bf6` (int:`329800735698586629295641978511506172918`) illustrating step by step the algorithms described in {{encoding}}{: format="title"} and {{decoding}}{: format="title"}.

## Bit Shifting Example {#bitShiftExample}
The bit shifting algorithm from {{bitShifting}} is used to remove the version and variant bits, and to shift the remaining 120 bits to be contiguous, so that they can be encoded with the Base32, Base58, or Base64 algorithms.

| Step | Operation | Result |
|------|-----------|--------|
| 1 | UUID int to 16-byte big-endian | `f81d4fae7dec11d0a76500a0c91e6bf6` |
| 2 | Unpack as 4 × uint32 BE | `[0xf81d4fae, 0x7dec11d0, 0xa76500a0, 0xc91e6bf6]` |
| 3 | Extract version `(ints[1] & 0xf000) >> 12` | **1** to bookend `B`/`b` (UUIDv1) |
| 4 | Extract variant `(ints[2] & 0xf0000000) >> 24` | **0xa0** to nybble `0xa` → bookend `K`/`k` |
| 5 | Rebuild `ints[1]`: close version gap, pull 4 bits from `ints[2]` | `0x7dec1d07` |
| 6 | Rebuild `ints[2]`: close variant gap, pull 8 bits from `ints[3]` | `0x6500a0c9` |
| 7 | Rebuild `ints[3]`: shift left 8, OR in variant byte | `0x1e6bf6a0` |
| 8 | Pack back to binary | `f81d4fae7dec1d076500a0c91e6bf6a0`, version=1 (`B`/`b`) |
{: title='UUID-NCName Bit Shifting Example for RFC9562, Section 4'}

The shifted 128-bit binary now has the 120 data bits contiguous (version and variant gaps removed), with the variant byte tucked into the trailing position.

## Encoding Example
The encoding algorithms from {{encoding}} are applied to the shifted binary from the previous example to produce the UUID-NCName representations found in {{genericTestVector}}.

### UUID-NCName-32 Example {#uuidNcname32Example}

| Step | Operation | Value |
|------|-----------|-------|
| 1 | Last octet `0xa0` >> 1 | `0x50` → bin becomes `...f650` |
| 2 | Base32 encode | `7AOU7LT55QOQOZIAUDER427WKA======` |
| 3 | Truncate to 25 chars | `7AOU7LT55QOQOZIAUDER427WK` |
| 4 | Version → Base32 char | 1 → `B` |
| 5 | Prepend version, lowercase | **`b7aou7lt55qoqoziauder427wk`** (26 chars) |
{: title='UUID-NCName-32 Example for RFC9562, Section 4'}

### UUID-NCName-58 Example {#uuidNcname58Example}

| Step | Operation | Value |
|------|-----------|-------|
| 1 | Remove last octet → variant | `0xa0` |
| 2 | variant `0xa0` >> 4 → Base32 char | 10 → `K` |
| 3 | Base58 encode remaining 15 bytes | `7wc88dU4e3NyJEj3e944D` (21 chars) |
| 4 | Pad to 21 chars with `_` | Already 21, no padding needed |
| 5 | Version → Base32 char | 1 → `B` |
| 6 | Concatenate ver + b58 + var | **`B7wc88dU4e3NyJEj3e944DK`** (23 chars) |
{: title='UUID-NCName-58 Example for RFC9562, Section 4'}

The following table illustrates the same steps for the UUIDv7 example from {{RFC9562, Section A.6}}: `017F22E2-79B0-7CC3-98C4-DC0C0C07398F`, whose shifted binary is `017f22e279b0cc38c4dc0c0c07398f90`. This example demonstrates underscore (_) padding and its application for Base58 encodings that are not 21 characters long.

| Step | Operation | Value |
|------|-----------|-------|
| 1 | Remove last octet → variant | `0x90` |
| 2 | variant `0x90` >> 4 → Base32 char | 9 → `J` |
| 3 | Base58 encode remaining 15 bytes | `3RrXaX7uTM6qdwrXwpC6` (20 chars) |
| 4 | Pad to 21 chars with `_` | `3RrXaX7uTM6qdwrXwpC6_` (1 underscore appended) |
| 5 | Version → Base32 char | 7 → `H` |
| 6 | Concatenate ver + b58 + var | **`H3RrXaX7uTM6qdwrXwpC6_J`** (23 chars) |
{: title='UUID-NCName-58 Example with Padding for RFC9562, Section A.6'}

### UUID-NCName-64 Example {#uuidNcname64Example}

| Step | Operation | Value |
|------|-----------|-------|
| 1 | Last octet `0xa0` >> 2 | `0x28` → bin becomes `...f628` |
| 2 | Base64url encode | `-B1Prn3sHQdlAKDJHmv2KA==` |
| 3 | Truncate to 21 chars | `-B1Prn3sHQdlAKDJHmv2K` |
| 4 | Version → Base32 char | 1 → `B` |
| 5 | Prepend version | **`B-B1Prn3sHQdlAKDJHmv2K`** (22 chars) |
{: title='UUID-NCName-64 Example for RFC9562, Section 4'}

## Decoding Example {#decodingExample}

Given the examples from the previous sections, decoding each can be illustrated as follows.

UUID-NCName-32: `b7aou7lt55qoqoziauder427wk`

| Step | Operation | Value |
|------|-----------|-------|
| 1 | Length = 26 | Base32 Encoding |
| 2 | Remove first char → version | `b` → 1 |
| 4b | Append padding `A======` | `7AOU7LT55QOQOZIAUDER427WKA======` |
| 4c | Base32 decode | `f81d4fae7dec1d076500a0c91e6bf650` |
| 4d | Last octet `0x50` << 1 | `0xa0` → shifted binary `...f6a0` |
{: title='UUID-NCName-32 Decode Example for RFC9562, Section 4'}

UUID-NCName-58: `B7wc88dU4e3NyJEj3e944DK`

| Step | Operation | Value |
|------|-----------|-------|
| 1 | Length = 23 | Base58 Encoding |
| 2 | Remove first char → version | `B` → 1 |
| 3a | Remove last char → variant | `K` → 10 (`0xa`) |
| 3b | variant << 4 | `0xa0` |
| 3c | Strip trailing underscores | `7wc88dU4e3NyJEj3e944D` (none removed) |
| 3d | Base58 decode (15 bytes) | `f81d4fae7dec1d076500a0c91e6bf6` |
| 3e | Append variant octet `0xa0` | shifted binary `...f6a0` |
{: title='UUID-NCName-58 Decode Example for RFC9562, Section 4'}

UUID-NCName-64: `B-B1Prn3sHQdlAKDJHmv2K`

| Step | Operation | Value |
|------|-----------|-------|
| 1 | Length = 22 | Base64 Encoding |
| 2 | Remove first char → version | `B` → 1 |
| 4a | Last char `K` already uppercase | no change |
| 4b | Append padding `A==` | `-B1Prn3sHQdlAKDJHmv2KA==` |
| 4c | Base64url decode | `f81d4fae7dec1d076500a0c91e6bf628` |
| 4d | Last octet `0x28` << 2 | `0xa0` → shifted binary `...f6a0` |
{: title='UUID-NCName-64 Decode Example for RFC9562, Section 4'}

All three produce shifted binary `f81d4fae7dec1d076500a0c91e6bf6a0` with version=1.

| Step | Operation | Value |
|------|-----------|-------|
| 5 | `version &= 0xf` | 1 |
| 6 | Unpack 4 × uint32 | `[0xf81d4fae, 0x7dec1d07, 0x6500a0c9, 0x1e6bf6a0]` |
| 7 | `variant = (ints[3] & 0xf0) << 24` | `0xa0000000` |
| 8 | `ints[3] >>= 8` | `0x001e6bf6` |
| 9 | `ints[3] \|= (ints[2] & 0xff) << 24` | `0xc91e6bf6` |
| 10 | `ints[2] >>= 8` | `0x006500a0` |
| 11 | `ints[2] \|= (ints[1] & 0xf) << 24 \| variant` | `0xa76500a0` |
| 12 | Restore version into `ints[1]` | `0x7dec11d0` |
| 13 | Pack to binary | `f81d4fae7dec11d0a76500a0c91e6bf6` |
| 14 | Format UUID | **`f81d4fae-7dec-11d0-a765-00a0c91e6bf6`** |
{: title='UUID-NCName Reverse Bit Shifting Example for RFC9562, Section 4'}

# Test Vectors {#testVectors}

The test vectors use the same UUIDs and illustrative examples as RFC9562 to illustrate the transformation of UUIDs to UUID-NCName representations.

## Generic UUID {#genericTestVector}

| Encoding               | Output                                 |
|------------------------|----------------------------------------|
| {{RFC9562, Section 4}} | `f81d4fae-7dec-11d0-a765-00a0c91e6bf6` |
| UUID-NCName-32         | `b7aou7lt55qoqoziauder427wk`           |
| UUID-NCName-58         | `B7wc88dU4e3NyJEj3e944DK`              |
| UUID-NCName-64         | `B-B1Prn3sHQdlAKDJHmv2K`               |
{: title='UUID-NCName Test Vectors for RFC9562, Section 4'}

## NIL UUID {#nilTestVectors}

| Encoding                 | Output                                 |
|--------------------------|----------------------------------------|
| {{RFC9562, Section 5.9}} | `00000000-0000-0000-0000-000000000000` |
| UUID-NCName-32           | `aaaaaaaaaaaaaaaaaaaaaaaaaa`           |
| UUID-NCName-58           | `A111111111111111______A`              |
| UUID-NCName-64           | `AAAAAAAAAAAAAAAAAAAAAA`               |
{: title='UUID-NCName Test Vectors for RFC9562, Section 5.9'}

## MAX UUID {#maxTestVectors}

| Encoding                 | Output                                 |
|--------------------------|----------------------------------------|
| {{RFC9562, Section 5.10}} | `FFFFFFFF-FFFF-FFFF-FFFF-FFFFFFFFFFFF` |
| UUID-NCName-32           | `p777777777777777777777777p`           |
| UUID-NCName-58           | `P8AQGAut7N92awznwCnjuQP`              |
| UUID-NCName-64           | `P____________________P`               |
{: title='UUID-NCName Test Vectors for RFC9562, Section 5.10'}

## UUIDv1

| Encoding                 | Output                                 |
|--------------------------|----------------------------------------|
| {{RFC9562, Section A.1}} | `C232AB00-9414-11EC-B3C8-9F6BDECED846` |
| UUID-NCName-32           | `byizkwaeucqpmhse7nppm5wcgl`           |
| UUID-NCName-58           | `B6S7oX73gv2Y1iTENdXX8hL`              |
| UUID-NCName-64           | `BwjKrAJQUHsPIn2vezthGL`               |
{: title='UUID-NCName Test Vectors for RFC9562, Section A.1'}

## UUIDv2

| Encoding                 | Output                                 |
|--------------------------|----------------------------------------|
| DCE Security             | `000003e8-cbb9-21ea-b201-00045a86c8a1` |
| UUID-NCName-32           | `caaaah2glxepkeaiaarninsfbl`           |
| UUID-NCName-58           | `C11KtP6Y9P3rRkvh2N1e__L`              |
| UUID-NCName-64           | `CAAAD6Mu5HqIBAARahsihL`               |
{: title='UUID-NCName Test Vectors for DCE Security'}

## UUIDv3

| Encoding                 | Output                                 |
|--------------------------|----------------------------------------|
| {{RFC9562, Section A.2}} | `5df41881-3aed-3515-88a7-2f4a814cf09e` |
| UUID-NCName-32           | `dlx2braj25vivrjzpjkauz4e6i`           |
| UUID-NCName-58           | `D3dTNMAmevR4NFAakRDtLdI`              |
| UUID-NCName-64           | `DXfQYgTrtUVinL0qBTPCeI`               |
{: title='UUID-NCName Test Vectors for RFC9562, Section A.2'}

## UUIDv4

| Encoding                 | Output                                 |
|--------------------------|----------------------------------------|
| {{RFC9562, Section A.3}} | `919108f7-52d1-4320-9bac-f847db4148a8` |
| UUID-NCName-32           | `esgiqr52s2ezaxlhyi7nucsfij`           |
| UUID-NCName-58           | `E55CtqYNqva1mcmaa877eoJ`              |
| UUID-NCName-64           | `EkZEI91LRMgus-EfbQUioJ`               |
{: title='UUID-NCName Test Vectors for RFC9562, Section A.3'}

## UUIDv5

| Encoding                 | Output                                 |
|--------------------------|----------------------------------------|
| {{RFC9562, Section A.4}} | `2ed6657d-e927-568b-95e1-2665a8aea6a2` |
| UUID-NCName-32           | `ff3lgk7pje5ullyjgmwuk5jvcj`           |
| UUID-NCName-58           | `F2K15VFLUBD326h169SNPjJ`              |
| UUID-NCName-64           | `FLtZlfeknaLXhJmWorqaiJ`               |
{: title='UUID-NCName Test Vectors for RFC9562, Section A.4'}

## UUIDv6

| Encoding                 | Output                                 |
|--------------------------|----------------------------------------|
| {{RFC9562, Section A.5}} | `1EC9414C-232A-6B00-B3C8-9F6BDECED846` |
| UUID-NCName-32           | `gd3euctbdfkyahse7nppm5wcgl`           |
| UUID-NCName-58           | `GrxRCnDiX4mxSq8bFQjT3_L`              |
| UUID-NCName-64           | `GHslBTCMqsAPIn2vezthGL`               |
{: title='UUID-NCName Test Vectors for RFC9562, Section A.5'}

## UUIDv7

| Encoding                 | Output                                 |
|--------------------------|----------------------------------------|
| {{RFC9562, Section A.6}} | `017F22E2-79B0-7CC3-98C4-DC0C0C07398F` |
| UUID-NCName-32           | `haf7sfytzwdgdrrg4bqgaoompj`           |
| UUID-NCName-58           | `H3RrXaX7uTM6qdwrXwpC6_J`              |
| UUID-NCName-64           | `HAX8i4nmwzDjE3AwMBzmPJ`               |
{: title='UUID-NCName Test Vectors for RFC9562, Section A.6'}

## UUIDv8

| Encoding                 | Output                                 |
|--------------------------|----------------------------------------|
| {{RFC9562, Section B.1}} | `2489E9AD-2EE2-8E00-8EC9-32D5F69181C0` |
| UUID-NCName-32           | `iese6tljo4lqa5sjs2x3jdaoai`           |
| UUID-NCName-58           | `I22HpMAy5M181AjPFG7eLXI`              |
| UUID-NCName-64           | `IJInprS7i4A7JMtX2kYHAI`               |
{: title='UUID-NCName Test Vectors for RFC9562, Section B.1'}

| Encoding                 | Output                                 |
|--------------------------|----------------------------------------|
| {{RFC9562, Section B.2}} | `5c146b14-3c52-8afd-938a-375d0df1fbf6` |
| UUID-NCName-32           | `ilqkgwfb4kkx5hcrxlug7d67wj`           |
| UUID-NCName-58           | `I3aR2J7aw1BJj4jJvfuWTXJ`              |
| UUID-NCName-64           | `IXBRrFDxSr9OKN10N8fv2J`               |
{: title='UUID-NCName Test Vectors for RFC9562, Section B.2'}

# Implementations {#implementations}

As of this writing, there are multiple implementations of UUID-NCName:

{: spacing="compact"}
- Perl, [](https://metacpan.org/pod/Data::UUID::NCName)
- Ruby, [](https://rubygems.org/gems/uuid-ncname)
- Java, by Werner Randelshofer [](https://github.com/wrandelshofer/UuidNCName)
- Dart, by Yulian Kuncheff, [](https://github.com/daegalus/uuid-format-tester)
