# F# RFC FS-1343 - `float64` abbreviation and `d`/`D` literal suffix

This RFC is split from section **e** of [the original literal-inference RFC](https://github.com/fsharp/fslang-design/pull/800), as requested by [the design review](https://github.com/fsharp/fslang-design/pull/800#issuecomment-3852354596).

The change is independent of target-typed literals: it adds an explicit spelling for the existing `System.Double` representation.

- [x] Approved in principle
- [ ] Implementation
- [ ] [Discussion](https://github.com/fsharp/fslang-design/discussions/FILL-ME-IN)

# Summary

Add:

1. `float64` as an FSharp.Core type abbreviation for `System.Double`, identical to the existing `float` and `double` abbreviations; and
2. `d` and `D` as suffixes for ordinary decimal-form `System.Double` literals.

```fsharp
let x: float64 = 1d
let y = 1.25D

[<Measure>]
type s

let elapsed: float64<s> = 0.5d<s>
```

The following types are identical:

```fsharp
type A = float
type B = double
type C = float64
```

Likewise, these literals denote the same `System.Double` value:

```fsharp
1.0
1.0d
1.0D
```

The existing `f`/`F` suffix continues to denote `float32`, and `m`/`M` continues to denote `decimal`.

# Motivation

F# exposes fixed-width names for most primitive numeric representations:

- `int32` and `int64`;
- `uint32` and `uint64`;
- `float32`.

The 64-bit IEEE-754 representation is instead normally named `float`, with `double` as a second abbreviation. Both are established and remain fully supported, but neither completes the fixed-width naming family. `float64` makes representation width explicit and improves symmetry in APIs, teaching material, code generation, and generic numeric code.

F# also has an explicit suffix for single-precision literals:

```fsharp
1.0f // float32
```

but no suffix that explicitly states “64-bit floating point”. An unsuffixed floating literal is already `float`, so the new suffix is not needed for inference. It is useful for:

- symmetry with `f`/`F`;
- generated code that emits a suffix for every primitive representation;
- making precision visually explicit next to `float32` values;
- writing an integer-shaped token such as `1d` while explicitly selecting floating-point representation;
- preserving intent during refactoring if surrounding target information changes.

# Detailed design

## `float64` type abbreviation

FSharp.Core adds a public type abbreviation:

```fsharp
type float64 = System.Double
```

It is identical to the existing abbreviations:

```fsharp
type float = System.Double
type double = System.Double
```

The measured form is also available with the same behavior as `float<'Measure>`:

```fsharp
[<MeasureAnnotatedAbbreviation>]
type float64<[<Measure>] 'Measure> = float<'Measure>
```

This is a transparent abbreviation, not a new CLI type. It adds no conversion, representation, reflection identity, equality rule, operator, or generic-math behavior.

Examples:

```fsharp
let distance: float64 = 42.0
let same (x: float) : float64 = x

[<Measure>]
type m

let length: float64<m> = 2.5<m>
```

`typeof<float64>` is `typeof<System.Double>`, and public signatures compiled with `float64` expose `System.Double` to other CLI languages.

## `d` and `D` suffixes

The lexical grammar gains a decimal-form double token:

```fsgrammar
token ieee64-suffixed =
    | (float | int) [Dd]
```

where `float` and `int` are the existing decimal numeric productions. The suffix is case-insensitive, matching `f`/`F` and `m`/`M`.

Valid examples include:

```fsharp
0d
1D
1.0d
6.022e23D
1_000.25d
```

The suffix is part of the numeric token, so normal adjacent-prefix handling applies:

```fsharp
-1d
+1.5D
```

A suffixed token is parsed, rounded, checked, and emitted exactly like the equivalent existing unsuffixed `float` literal:

```fsharp
let a = 0.1
let b = 0.1d

// a and b have the same type, value, and emitted representation.
```

The suffix can be combined with units of measure wherever the existing numeric-literal grammar permits a measure:

```fsharp
[<Measure>]
type kg

let mass = 12d<kg>
```

It is also available in the same constant contexts as existing `float` literals, including constant patterns and explicitly valid `[<Literal>]` definitions.

## Hexadecimal bit-pattern literals

The existing `LF` spelling for an IEEE-754 binary64 bit pattern is unchanged:

```fsharp
let negativeZero = 0x8000000000000000LF
```

`d` and `D` are hexadecimal digits, so spellings such as `0x1d` and `0x1D` remain hexadecimal integer literals with value 29. This RFC does not introduce `0x...d` as a second bit-pattern form.

## Name and display conventions

`float`, `double`, and `float64` are interchangeable source-level abbreviations. Existing compiler and tooling output may continue to use `float` as the canonical display name to avoid churn in inferred signatures, diagnostics, and generated documentation.

Code completion and symbol documentation should nevertheless list `float64`, and source annotations written as `float64` remain valid and navigable.

This RFC does not deprecate or discourage `float` or `double`.

# Changes to the F# spec

## Basic types

Add `float64` to the set of FSharp.Core primitive type abbreviations and describe it as identical to `float`, `double`, and `System.Double`.

Add the measure-annotated abbreviation corresponding to `float<'Measure>`.

## Lexical analysis

Add:

```fsgrammar
token ieee64-suffixed =
    | (float | int) [Dd]
```

and include it among simple constant expressions with type `float`/`System.Double`.

Clarify that `xint` consumes `d` and `D` as hexadecimal digits, so hexadecimal integer and existing `LF` forms are unaffected.

## Type checking and code generation

No new typing rule is required beyond assigning `System.Double` to the new token. Reuse the existing parser, constant representation, rounding, overflow, units-of-measure, quotation, pattern, and code-generation behavior for `ieee64` literals.

# Drawbacks

## Three names for one type

F# will have `float`, `double`, and `float64` for `System.Double`. Additional aliases add vocabulary and may lead projects to adopt different style conventions.

## The suffix is redundant for ordinary inference

`1.0` already has type `float`. The suffix adds explicitness and symmetry rather than new expressive power for floating-shaped tokens.

## `d` differs from C#'s decimal conventions only by ecosystem expectation

C# uses `d` for `double` and `m` for `decimal`, so the proposal aligns with C#. Some users may nevertheless initially read `d` as “decimal”. F# continues to reserve `m`/`M` for decimal literals, and documentation should call this out.

## A new automatically available type name can collide

As with any FSharp.Core name addition, unqualified `float64` may affect name resolution in code that also opens or defines another entity with that name. Local definitions and qualified names retain the normal F# shadowing and disambiguation mechanisms.

# Alternatives

## Keep only `float` and `double`

This avoids another alias but leaves `float32` without a fixed-width counterpart and makes representation-oriented APIs less regular.

## Add only `float64`

This supplies the most useful naming symmetry but leaves literals asymmetric with `f`/`F`. The suffix is small and independently explicit, so this RFC includes both approved pieces.

## Add only `d`/`D`

This allows explicit literals but does not solve the fixed-width type-name inconsistency.

## Use a different suffix

`l`/`L` already belongs to integer and binary floating-point forms, and a longer suffix such as `f64` would introduce a new multi-character convention. `d`/`D` is concise, currently reserved in the relevant decimal literal position, and familiar from C# and other .NET languages.

## Make `float64` the canonical printed name

Changing inferred signature and diagnostic output from `float` to `float64` would create broad textual churn without changing semantics. Existing canonical rendering should remain stable.

# Prior art

- C# and Java use `d`/`D` for an explicitly double-precision literal.
- Rust uses the fixed-width type name `f64` and an `f64` literal suffix.
- F# already provides `float32`, `int32`, `int64`, and corresponding explicit literal suffixes for many primitive representations.
- F# already has multiple transparent names for the same CLI type, including `float`/`double`, `float32`/`single`, and `int`/`int32`.

# Compatibility

## Source compatibility

The new suffixed spellings are currently reserved/invalid decimal literal forms, so making them valid does not change the meaning of a previously valid numeric token.

Hexadecimal literals such as `0x1d` are explicitly unchanged.

Adding the `float64` abbreviation can introduce the ordinary name-resolution collision described under Drawbacks, but does not change the identity of any existing type or expression.

## Binary compatibility

There is no new binary type. `float64` and `d`/`D` compile to `System.Double` exactly as `float`, `double`, and unsuffixed IEEE-64 literals do today.

## Older compilers

An older compiler rejects source containing `1d` or `1D`. It can consume assemblies built from such source because the emitted constants and signatures use ordinary `System.Double`.

An older compiler using an FSharp.Core version that contains the `float64` abbreviation may consume compiled signatures in the same way as other transparent abbreviations; cross-language consumers see only `System.Double`.

## Language version

The new literal token should initially be enabled under the corresponding preview language version. The FSharp.Core abbreviation is delivered with the matching FSharp.Core release.

# Interop

All three F# type names map to `System.Double`. Public APIs authored with `float64` are indistinguishable in CLI metadata from APIs authored with `float` or `double`, and callers in C#, Visual Basic, or other CLI languages require no changes.

The `d`/`D` spelling aligns F# source more closely with C# source generators and examples while retaining F#'s existing runtime representation.

# Pragmatics

## Diagnostics

An out-of-range or malformed `d`/`D` literal uses the same diagnostic family and source range as an equivalent existing `float` literal.

Tooling should not suggest `d` for decimal; completion/documentation should state that `d`/`D` denotes `System.Double` and `m`/`M` denotes `System.Decimal`.

## Tooling

- Syntax highlighting classifies `d`/`D` as part of the numeric literal.
- Hover reports `float` or `float64` according to the tool's existing abbreviation-display policy.
- Completion offers `float64` alongside `float`, `double`, and `float32`.
- Rename does not treat a primitive abbreviation as a user declaration.

## Performance

No runtime conversion or additional instruction is introduced. Parsing and constant emission reuse the existing binary64 implementation.

## Scaling

The feature adds one primitive alias and one lexical suffix pair. It does not alter overload resolution, inference, constraint solving, or generic code size.

## Culture-aware formatting/parsing

Literal parsing remains culture-invariant. The decimal separator is `.`, exponents use `e`/`E`, and the generated value is independent of the compiler process culture.

# Unresolved questions

- Should formatter style options be able to prefer `float64` over `float` in explicit annotations, while leaving inferred displays unchanged?
- Should documentation recommend lowercase `d` as the conventional spelling, matching lowercase `f` and `m`?
