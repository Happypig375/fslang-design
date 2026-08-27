# F# RFC FS-1345 - UTF-8 `B`-suffix string literals

The design suggestion [Extending `B` string suffix to be UTF-8 strings](https://github.com/fsharp/fslang-suggestions/issues/1421) has been marked "approved in principle".

This RFC is the focused successor to section **r** of [the original omnibus RFC](https://github.com/fsharp/fslang-design/pull/800), as requested by [the design review](https://github.com/fsharp/fslang-design/pull/800#issuecomment-3852354596).

- [x] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/1421)
- [x] Approved in principle
- [ ] Implementation
- [ ] [Discussion](https://github.com/fsharp/fslang-design/discussions/FILL-ME-IN)

# Summary

Extend existing `B`-suffix string literals from ASCII-only byte arrays to compile-time UTF-8 byte arrays.

```fsharp
let ascii = "hello"B
let chinese = "你好"B
let escaped = "\u4F60\u597D"B
let path = @"C:\資料"B
```

All values have type `byte array`. The compiler decodes the source literal according to its existing regular or verbatim string rules, validates the resulting Unicode text, and encodes it as UTF-8 at compile time.

For example:

```fsharp
"é"B
```

produces a fresh byte array containing:

```fsharp
[| 0xC3uy; 0xA9uy |]
```

ASCII literals are unchanged byte-for-byte.

This RFC does not change ordinary `string` literals, does not add target typing, does not add a null terminator or byte-order mark, and does not change the existing one-byte character literal form such as `'A'B`.

# Motivation

F# already has a convenient byte-array literal:

```fsharp
"Content-Type: "B
```

Its current language definition permits only ASCII characters. That limitation reflects neither the name of the type (`byte array`) nor modern network and file formats, where UTF-8 is the dominant text encoding.

Without this change, a non-ASCII compile-time byte sequence requires one of two unattractive forms:

```fsharp
// Runtime encoding and allocation machinery.
System.Text.Encoding.UTF8.GetBytes("你好")

// Manual bytes, efficient but hard to read and maintain.
[| 0xE4uy; 0xBDuy; 0xA0uy; 0xE5uy; 0xA5uy; 0xBDuy |]
```

Extending the existing suffix provides:

- readable protocol, file-format, parser, and test constants;
- compile-time validation and encoding;
- no runtime text encoder invocation;
- exact, deterministic bytes;
- a natural evolution of syntax already dedicated to byte arrays.

The feature is independent of the broader target-typed string proposal rejected in the original RFC review. A normal string remains `System.String`, while `"..."B` remains an explicit `byte array` literal.

# Detailed design

## Eligible literal forms

The change applies to the two existing string-like byte-array token forms:

```fsharp
"regular string"B
@"verbatim string"B
```

Examples:

```fsharp
let regular = "γειά\n"B
let verbatim = @"C:\資料\file.txt"B
```

The suffix remains uppercase `B`, matching existing F# syntax.

This RFC does not add `B` to:

- interpolated strings;
- triple-quoted strings;
- format strings or printf format values;
- character literals beyond their current behavior.

Those forms can be considered separately if independently motivated.

## Decoding and encoding pipeline

A `B`-suffix string is processed in three conceptual stages.

### 1. Decode the source literal

The compiler applies the existing rules for the literal kind:

- regular strings process escape sequences, trigraphs, Unicode escapes, and line continuation;
- verbatim strings preserve backslashes and newlines and process doubled quotation marks according to existing rules.

This produces the same sequence of UTF-16 code units that the corresponding ordinary F# `string` literal would contain.

For example, these produce the same bytes:

```fsharp
"你好"B
"\u4F60\u597D"B
```

Decimal and Unicode escape notation denotes Unicode text before UTF-8 encoding:

```fsharp
"\250"B // U+00FA, encoded as C3 BA
```

It does not mean “insert the raw byte 250”. A raw byte sequence should be written as a byte-array expression.

### 2. Validate Unicode well-formedness

The resulting UTF-16 sequence must contain only valid Unicode scalar values:

- a high surrogate must be followed by a low surrogate;
- a low surrogate must be preceded by a high surrogate;
- values outside the Unicode scalar range are rejected by the existing escape parser.

An unpaired surrogate is a compile-time error:

```fsharp
let invalid = "\uD800"B
//             ^ error: malformed Unicode text in UTF-8 byte string
```

Validation uses replacement-disabled semantics. The compiler never silently inserts U+FFFD.

### 3. Encode as UTF-8

Each Unicode scalar value is encoded using standard UTF-8:

- U+0000 through U+007F: one byte;
- U+0080 through U+07FF: two bytes;
- U+0800 through U+FFFF, excluding surrogates: three bytes;
- U+10000 through U+10FFFF: four bytes.

The encoding uses the shortest valid form and is deterministic across compiler platforms.

No Unicode normalization is performed. Canonically equivalent source strings can therefore produce different byte sequences, exactly as `Encoding.UTF8.GetBytes` does for the corresponding .NET strings:

```fsharp
"é"B       // U+00E9
"e\u0301"B // U+0065 U+0301
```

## Result type and value

The type remains:

```fsharp
byte array
```

The array contains exactly the encoded payload:

- no UTF-8 byte-order mark is prepended;
- no null terminator is appended;
- the array length is the number of encoded bytes, not the number of UTF-16 code units or Unicode scalar values.

```fsharp
let bytes = "😀"B
bytes.Length = 4
```

## Mutability and allocation identity

Existing `B`-literal array semantics are preserved. Each evaluation produces a distinct mutable array:

```fsharp
let make () = "abc"B

let a = make ()
let b = make ()

a.[0] <- 0uy

// b.[0] remains 0x61uy
```

The compiler may store encoded bytes in static data and copy them into a new array, or emit equivalent element initialization. It must not expose shared mutable static storage as the result.

This differs intentionally from C# `u8` literals, whose natural type is `ReadOnlySpan<byte>` and whose backing data may be shared.

## ASCII compatibility

For a literal containing only U+0000 through U+007F, UTF-8 uses one byte per character with the same numeric value. Therefore every currently valid ASCII `B` literal produces exactly the same array as before:

```fsharp
"A\000\n"B
```

Existing escape interpretation, line continuation, array freshness, and type are unchanged.

## Existing non-ASCII diagnostics

Current compilers diagnose non-ASCII `B` literals because the language restricts them to one-byte ASCII characters. Under the new language version, those diagnostics are replaced by UTF-8 validation:

```fsharp
"ú"B       // valid, C3 BA
"\128"B   // valid U+0080, C2 80
"\u0080"B // valid U+0080, C2 80
```

Source that suppressed the old out-of-spec warning and relied on a legacy one-byte value may observe different bytes under the new language version. Such code can preserve raw-byte intent explicitly:

```fsharp
[| 0x80uy |]
```

Language-version gating keeps the old behavior available when compiling under an older version.

## Byte character literals

The existing byte-character form remains unchanged:

```fsharp
'A'B // byte value 0x41
```

A character literal denotes exactly one `byte`, not a byte sequence. Since every non-ASCII Unicode scalar requires multiple UTF-8 bytes, non-ASCII byte-character literals remain invalid:

```fsharp
'é'B // error
```

This distinction keeps the type and cardinality of `'x'B` stable.

## Constant and pattern contexts

A `B` string produces a mutable array and is not a CLI constant. This RFC does not make it valid as a `[<Literal>]` value, optional-argument default, or constant pattern.

Existing quotations and typed syntax should represent the value as an ordinary byte-array construction with the compile-time encoded elements, preserving fresh-array semantics.

# Changes to the F# spec

## Lexical analysis

Retain the existing token grammar:

```fsgrammar
token bytearray = " string-char * "B
token verbatim-bytearray = @" verbatim-string-char * "B
```

Replace the semantic restriction that byte-array strings contain only characters with encodings no greater than 127.

Specify that the decoded Unicode contents are validated as well-formed UTF-16 and encoded as UTF-8 without BOM, terminator, replacement, or normalization.

Keep `bytechar` restricted to a single ASCII byte.

## Simple constant expressions

Clarify that a byte-array string is an array-valued literal expression rather than a CLI constant, as today. Its element sequence is the UTF-8 encoding determined at compile time.

## Code generation

Require a fresh `byte array` on every evaluation and permit any lowering with equivalent bytes, evaluation behavior, mutability, and identity.

## Language version

Define old-version behavior as the existing ASCII-only rules and new-version behavior as this RFC's UTF-8 rules.

# Drawbacks

## A suffix historically associated with ASCII gains broader meaning

Some users may have understood `B` as “ASCII byte string” rather than “byte-array string”. The syntax itself says only `B`, and UTF-8 is an ASCII-compatible extension, but the conceptual change should be documented.

## Unicode escape values no longer correspond to raw byte values

A spelling such as `"\250"B` becomes UTF-8 for U+00FA, not a one-byte value `0xFA`. This follows string semantics and Unicode correctness, but raw protocol bytes must be written as numeric byte arrays.

## Mutable arrays still allocate

Compile-time encoding removes encoder cost, but each evaluation must preserve fresh mutable-array identity. APIs expecting a read-only span may still require conversion or a separate target-typed collection feature.

## Interpolated and triple-quoted forms remain asymmetric

Only currently existing `B` string forms are extended. Users cannot yet write an interpolated UTF-8 byte literal or a triple-quoted `B` literal.

## Malformed-surrogate validation adds a diagnostic case

Ordinary .NET strings can contain unpaired UTF-16 surrogates. UTF-8 byte literals reject them rather than applying replacement fallback, because source literals should produce deterministic valid UTF-8.

# Alternatives

## Keep `B` ASCII-only

This preserves the historical restriction but forces runtime encoding or manually maintained bytes for non-ASCII constants.

## Add a new `u8` suffix

A new suffix could mirror C#, potentially with `ReadOnlySpan<byte>` semantics. F# already owns `B` syntax for byte arrays, and the approved proposal specifically evolves it. Reusing `B` is smaller and preserves its mutable-array type.

## Use the platform default encoding

Platform encodings are culture- and machine-dependent and cannot provide deterministic source semantics. UTF-8 is explicitly specified.

## Encode UTF-16 code units independently

Encoding individual surrogate code units would permit malformed output and differ from Unicode UTF-8. Validation and scalar-value encoding are required.

## Replace malformed sequences with U+FFFD

Replacement would hide source errors and make an invalid literal silently differ from its apparent text. Compile-time rejection is safer.

## Return `ReadOnlySpan<byte>`

That could avoid fresh-array allocation and align exactly with C# `u8`, but it would be a breaking type and mutability change for existing `B` literals. Target-typed or separately suffixed read-only UTF-8 literals can be proposed independently.

## Normalize text before encoding

Normalization can be useful in application protocols but changes user-provided text and has multiple valid policies. Literal encoding should be lossless and normalization-free.

# Prior art

- C# `u8` literals validate Unicode and encode UTF-8 at compile time, with `ReadOnlySpan<byte>` result semantics.
- Rust byte strings are byte-oriented and ASCII-restricted, while ordinary strings are UTF-8; this RFC instead evolves F#'s existing string-like byte-array syntax.
- Python and several systems languages provide explicit byte-string syntax, though their source-encoding and escape semantics differ.
- .NET's `UTF8Encoding` without BOM and with invalid-sequence exceptions provides the equivalent byte mapping for well-formed input.

# Compatibility

## Source compatibility

Every conforming ASCII `B` literal retains the same type, bytes, allocation identity, and behavior.

Non-ASCII `B` literals are outside the current specification and receive diagnostics in current compilers. They become valid UTF-8 literals under the new language version.

Code that suppressed an old warning and depended on legacy emitted one-byte values may change behavior; compiling under the older language version or writing an explicit numeric byte array preserves that behavior.

The meaning of ordinary strings and byte-character literals is unchanged.

## Binary compatibility

Compiled code exposes only ordinary `System.Byte[]` values and array initialization. No runtime helper, metadata marker, or FSharp.Core change is required.

## Older compilers

Older compilers continue to diagnose non-ASCII `B` source. They can reference assemblies produced by a new compiler because UTF-8 literals leave no new metadata representation.

## Language version

The UTF-8 interpretation should initially be gated by the corresponding preview language version. This is important because current implementations may emit legacy bytes for diagnosed-but-not-fatal non-ASCII source.

# Interop

The resulting value is `System.Byte[]`, directly consumable by .NET networking, parsing, cryptography, file, and interop APIs.

The payload matches standard UTF-8 produced by .NET, C#, Rust, and other conforming implementations for the same Unicode scalar sequence, except that F# exposes a fresh mutable array and does not include C#'s hidden trailing null byte.

No runtime dependency on `System.Text.Encoding` is introduced at the call site.

# Pragmatics

## Diagnostics

Diagnostics should identify:

- an unpaired high or low surrogate;
- an invalid Unicode escape;
- a non-ASCII byte-character literal, with guidance to use a string or explicit bytes;
- use of `B` on an unsupported string form.

Where possible, the source range should cover the offending escape or character rather than the entire literal.

The old “outside ASCII” warning is not emitted for valid new-version UTF-8 string literals.

## Tooling

- Syntax highlighting continues to classify `B` as a byte-string suffix.
- Hover reports `byte array` and may show the encoded byte length.
- IDE byte visualizers should display the exact UTF-8 payload, not reinterpret it using the current culture.
- A code action may convert a UTF-8 `B` literal to an explicit hexadecimal byte array for protocols that prefer visible bytes.

## Performance

Encoding is performed at compile time. Runtime work is limited to constructing the fresh array, using the same optimization freedom as existing byte-array literals.

Large literals increase assembly data and incur a copy per evaluation because of mutability. Hoisting a literal to a module-level binding retains one array instance exactly as ordinary F# binding semantics dictate.

## Scaling

Compilation cost and emitted data size are linear in the number of UTF-8 bytes. Validation requires a single pass over decoded UTF-16 and encoding requires a single pass or an equivalent combined implementation.

## Culture-aware formatting/parsing

The feature is culture-invariant. Unicode decoding, scalar validation, and UTF-8 encoding do not depend on current culture, code page, operating system, or locale.

# Unresolved questions

- Should triple-quoted `"""..."""B` literals be added in a follow-up for large UTF-8 payloads?
- Should interpolated UTF-8 byte strings be designed separately around builders or handlers rather than implicit object formatting?
- Should tooling offer a warning for very large `B` literals evaluated repeatedly, where a module-level cached binding may be preferable?
