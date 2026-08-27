# F# RFC FS-1342 - Contextual target typing for numeric and collection literals

The design suggestion [Type inference for literals](https://github.com/fsharp/fslang-suggestions/issues/1421) has been marked "approved in principle".

This RFC is the focused successor to [the original omnibus RFC](https://github.com/fsharp/fslang-design/pull/800). It contains only the two levels approved in [the design review](https://github.com/fsharp/fslang-design/pull/800#issuecomment-3852354596): literals at CLI method call sites and literals checked under an explicit type annotation.

- [x] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/1421)
- [x] Approved in principle
- [ ] Implementation
- [ ] [Discussion](https://github.com/fsharp/fslang-design/discussions/FILL-ME-IN)

# Summary

Allow an unsuffixed numeric literal or an eligible bracket collection expression to be checked directly against a concrete target type in either of these contexts:

1. an argument of a CLI method, constructor, indexer, or property setter; or
2. an expression covered by an explicit type annotation.

```fsharp
open System
open System.Collections.Immutable

type Texture =
    static member Upload(data: ReadOnlySpan<byte>, opacity: float32) =
        // ...
        ()

Texture.Upload([0; 64; 128; 255], 0.75)

let packetKind: byte = 3
let samples: ImmutableArray<byte> = [0; 64; 128; 255]
```

The feature is deliberately local and top-down:

```fsharp
let values = [0; 64; 128; 255] // int list, as today

// No target information flows backwards through `values`.
Texture.Upload(values, 0.75)    // still an error
```

It does **not**:

- change the default types of unannotated literals;
- infer from later uses, pipelines, local bindings, or function result use;
- introduce new SRTP constraints or generalized literal types;
- use a literal to infer an otherwise unknown generic type argument;
- retarget operands merely because an enclosing operator expression is annotated.

# Motivation

F# defaults unsuffixed integer literals to `int`, unsuffixed floating-point literals to `float`, and bracket expressions to F# lists. Those defaults are valuable for local reasoning, but they add ceremony at .NET API boundaries where the callee already declares a different representation.

Typical friction includes:

- numeric parameters such as `byte`, `uint32`, `float32`, or `decimal`;
- arrays, spans, immutable collections, and collection-builder types;
- repeated suffixes and explicit factories whose only purpose is to restate a method signature;
- avoidable intermediate F# lists when the API expects a different collection.

Current code often looks like:

```fsharp
Texture.Upload([| 0uy; 64uy; 128uy; 255uy |].AsSpan(), 0.75f)
```

The proposed form keeps the type visible in the API signature while retaining compile-time bounds checking:

```fsharp
Texture.Upload([0; 64; 128; 255], 0.75)
```

The restricted contexts are essential. They deliver the interop benefit without changing the meaning of literals in ordinary inference, inferred public signatures, or generic code.

# Detailed design

## Terminology

A **contextual literal** is an eligible literal checked with a **contextual target type**.

A contextual target type is **fully known** when it contains no unresolved inference variables relevant to the literal conversion. A constructed generic type may contain ordinary type parameters from the surrounding declaration, but the literal itself must not be used to choose those parameters.

A **current-language conversion** means any conversion already available before this RFC, including the conversions described by [FS-1093](../FSharp-6.0/FS-1093-additional-conversions.md).

## Contextual positions

### CLI call arguments

Contextual checking is available for arguments to:

- instance and static CLI methods;
- object constructors;
- indexers;
- property setters;
- delegate `Invoke` methods when the delegate signature is already known.

It applies to positional, named, and optional arguments.

The proposal does not extend ordinary curried F# function application in its first version:

```fsharp
let consumeByte (x: byte) = x

consumeByte 1          // unchanged in this RFC
consumeByte (1: byte)  // enabled by the explicit annotation rule
```

This boundary follows the approved scope of "method call sites". A later RFC may assess whether direct application of a non-generic F# function should participate.

For an overloaded member, each candidate supplies its own potential target type during candidate checking. The compatibility rules in [Overload resolution](#overload-resolution) determine when the new conversions may participate.

### Explicit type annotations

Contextual checking is available when an expression is checked under an explicit type annotation, including:

```fsharp
let x: byte = 1
let y = (0.5: float32)

let create (): decimal =
    19.95

let bytes: byte array =
    [0; 127; 255]
```

The permission propagates through expression forms that already pass a known result type directly to their immediate subexpressions, including parentheses and the result branches of `if`, `match`, and `try`:

```fsharp
let status: byte =
    if ready then 1 else 2
```

It does not propagate through an intervening local binding, lambda, function or member call, pipeline, computation expression, or operator application:

```fsharp
let ratio: float32 = 1 / 2
// The annotation does not choose the overload of (/), so this keeps current behavior.

let ratio2: float32 = (1.0: float32) / (2.0: float32)
```

### Generic member applications

A contextual literal is not evidence for generic type inference.

```fsharp
type C =
    static member Identity<'T>(value: 'T) = value

C.Identity(1) // 'T = int, as today
```

Contextual checking is available after the type argument has been fixed independently:

```fsharp
C.Identity<byte>(1)

type Pairing =
    static member Pair<'T>(marker: 'T, value: 'T) = marker, value

Pairing.Pair(0uy, 1) // the first argument fixes 'T = byte
```

For the first implementation, a candidate such as `M(ReadOnlySpan<'T>)` is not made applicable solely by inferring `'T` from the elements of the bracket expression.

## Ordering relative to existing conversions

Contextual literal conversions are last-resort conversions.

For a non-overloaded expected type:

1. check the expression with the current language, including FS-1093;
2. only if that check fails, and the expression and context are eligible, try contextual literal checking.

For overloaded calls, the same principle is implemented by the two-phase process described later.

This means an existing widening conversion remains the conversion used:

```fsharp
let x: int64 = 1
```

If this already succeeds by checking `1` as `int` and applying the current `int32 -> int64` conversion, that elaboration and its diagnostics remain unchanged. Contextual parsing as an `int64` literal is not substituted into an already valid program.

## Numeric literals

### Eligible syntax

Contextual numeric checking applies to unsuffixed integer-form and floating-point-form literals, including their existing decimal, binary, octal, hexadecimal, exponent, and underscore spellings.

An immediately adjacent unary `+` or `-` is considered together with the literal for representability:

```fsharp
let minimum: sbyte = -128
```

A suffix fixes the literal's current source type and disables contextual re-parsing of that literal:

```fsharp
let x: int64 = 1uy
// This remains a byte literal followed by any current-language conversion that applies.
```

### Direct numeric targets

The initial target set is limited to intrinsic F# numeric representations for which the compiler already has target-specific literal parsing and constant generation.

| Source form | Contextual targets | Existing default |
|---|---|---|
| Integer-form literal | `sbyte`, `byte`, `int16`, `uint16`, `int32`, `uint32`, `int64`, `uint64`, `nativeint`, `unativeint`, `bigint`, `float32`, `float`, `decimal` | `int32` |
| Floating-point-form literal | `float32`, `float`, `decimal` | `float` |

Transparent type abbreviations are normalized to their underlying intrinsic type.

The first version does not add direct contextual construction for:

- enum or nullable types;
- `Half`, `Int128`, `UInt128`, `NFloat`, `Complex`, or `Rune`;
- arbitrary `INumberBase<_>` implementations;
- `NumericLiteralX` modules;
- a multi-step search through a different numeric source type and `op_Implicit`.

Any current-language conversion that already supports one of those types remains available.

### Target-specific parsing and checking

The token is parsed directly according to the contextual target. It is not first emitted as `int` or `float` and then narrowed at runtime.

```fsharp
let ok: byte = 255
let error: byte = 256
//                    ^ compile-time error
```

The compiler reuses the parsing, rounding, overflow, underflow, and constant-generation behavior of the corresponding explicitly suffixed literal.

Consequently:

- integral values must be within the inclusive range of the target;
- unsigned targets reject negative values;
- `nativeint` and `unativeint` retain their existing platform-independent restrictions;
- floating-point and decimal values use the same rounding and overflow behavior as their current explicit forms;
- no runtime checked conversion is emitted merely to construct the literal.

### Units of measure

This RFC selects a numeric representation only. Existing units-of-measure inference and checking remain unchanged.

The conversion does not invent, erase, or change a measure. A literal with explicit measure syntax may be contextually represented when the source and target measures unify:

```fsharp
[<Measure>]
type px

let width: float32<px> = 1920.0<px>
```

Whether an unmeasured literal can inhabit a measured target continues to be decided by the existing units-of-measure rules.

### Literal definitions and patterns

An explicitly typed `[<Literal>]` binding may use contextual numeric checking when the target is otherwise a valid F#/CLI literal type:

```fsharp
[<Literal>]
let PacketKind: byte = 3
```

Without an annotation, defaulting is unchanged.

Numeric literal patterns are outside this RFC. Pattern conversion, exhaustiveness, and equality behavior require separate consideration, so existing explicit suffixes remain required where they are required today.

## Bracket collection expressions

### Eligible syntax

This RFC reinterprets a restricted subset of existing bracket-list syntax when a contextual collection target is available.

The eligible forms are:

- the empty expression `[]`;
- expression elements, with or without explicit `yield`;
- spread elements written with `yield!`.

```fsharp
[]
[1; 2; 3]
[yield 1; yield 2]
[0; yield! otherValues; 255]
```

These correspond to C# collection-expression elements and spread elements.

The following retain their current F# list-expression meaning in the first version:

- range syntax such as `[1 .. 10]`;
- `for`, `while`, `let`, `use`, `try`, `if`, or `match` statements at collection-expression level;
- other sequence-expression constructs whose lowering is not a simple sequence of element and spread operations.

They may still appear inside a parenthesized element expression. Extending contextual construction to the full sequence-expression grammar can be considered separately.

### Target types

The target categories follow .NET/C# collection-expression conventions:

1. an F# list, using current list-expression behavior;
2. a one-dimensional array `T array`;
3. `System.Span<T>`;
4. `System.ReadOnlySpan<T>`;
5. a type with a valid `System.Runtime.CompilerServices.CollectionBuilderAttribute`;
6. a class or struct that:
   - implements `System.Collections.IEnumerable`;
   - has an accessible applicable parameterless constructor; and
   - for a non-empty expression, has an accessible applicable instance or extension `Add` method accepting one element;
7. one of:
   - `System.Collections.Generic.IEnumerable<T>`;
   - `System.Collections.Generic.IReadOnlyCollection<T>`;
   - `System.Collections.Generic.IReadOnlyList<T>`;
   - `System.Collections.Generic.ICollection<T>`;
   - `System.Collections.Generic.IList<T>`.

There is no contextual conversion to a multidimensional array.

Nullable lifting and arbitrary interface synthesis beyond the listed interfaces are outside the first version.

### Element type and checking

Each eligible target determines one element type `T`.

- An expression element is checked directly against `T`.
- A spread element is checked as an iterable expression; its iteration type must have a current-language or contextual element conversion to `T`.
- Numeric literal elements may recursively use the numeric rules in this RFC.
- Nested bracket expressions may be contextual when their own target is fully known.

```fsharp
let bytes: byte array = [0; 127; 255]

let matrix: byte array array =
    [[1; 2]; [3; 4]]
```

The expression does not first construct an `int list`. Each element is checked for the final element type.

The empty expression is valid only when its target supplies an unambiguous element type.

All element and spread source expressions are evaluated exactly once from left to right. Following the C# collection-expression model, source expressions are evaluated before count queries and enumeration used to materialize the result.

### Collection builders

A collection target may identify a create method with `CollectionBuilderAttribute`.

The attribute's builder type and method name are validated using the .NET collection-expression shape:

- the builder type is a non-generic class or struct;
- the create method is declared directly on that builder type;
- it is static and accessible;
- its generic arity matches the target type's arity;
- it has exactly one by-value `ReadOnlySpan<E>` parameter;
- `E` is identical to the target's iteration type;
- its return type has an identity, reference, or boxing conversion to the target;
- exactly one applicable create method satisfies the shape.

Illustrative declaration:

```fsharp
open System
open System.Collections
open System.Collections.Generic
open System.Runtime.CompilerServices

[<CollectionBuilder(typeof<BufferBuilder>, "Create")>]
type Buffer<'T> internal (items: 'T array) =
    member _.Length = items.Length

    interface IEnumerable<'T> with
        member _.GetEnumerator() =
            (items :> seq<'T>).GetEnumerator()

    interface IEnumerable with
        member _.GetEnumerator() =
            (items :> IEnumerable).GetEnumerator()

and BufferBuilder =
    static member Create<'T>(items: ReadOnlySpan<'T>) : Buffer<'T> =
        Buffer(items.ToArray())
```

Usage:

```fsharp
let buffer: Buffer<byte> = [1; 2; 3]
```

Invalid or ambiguous builder metadata is diagnosed at the use site. The builder is invoked exactly once after its input elements have been evaluated.

### Constructible and interface targets

For a constructible target, the compiler follows collection-initializer semantics:

- construct the target once;
- use an accessible `capacity: int` constructor when the length is statically known and such a constructor is applicable, otherwise use the parameterless constructor;
- invoke the selected `Add` method for each element in order;
- enumerate each spread source in order and add each value.

For mutable interface targets (`ICollection<T>` and `IList<T>`), the result is a `List<T>`.

For non-mutable interface targets (`IEnumerable<T>`, `IReadOnlyCollection<T>`, and `IReadOnlyList<T>`), the implementation may use an existing or synthesized implementation, but the observable mutability and interface behavior must follow the C# collection-expression contract.

### Arrays, spans, and lowering freedom

Known-length and unknown-length expressions are both permitted.

The implementation may choose static data, inline storage, stack storage, a growable buffer, or a heap array, provided it preserves:

- the requested target type and mutability;
- left-to-right, exactly-once source evaluation;
- exception timing and observable side effects;
- span and byref-like escape safety;
- observable aliasing behavior.

This RFC does not guarantee stack allocation. It guarantees that contextual construction does not semantically require an intermediate F# list.

A `Span<T>` target is writable. A `ReadOnlySpan<T>` target may refer to immutable static data only when the values, target framework, and lifetime rules permit it.

## Overload resolution

Source compatibility is protected by two phases.

### Phase A: current language

Determine applicable candidates and the better member exactly as the current language does, with contextual literal checking disabled.

If Phase A contains any applicable candidate, the call is resolved entirely by current rules. A candidate made possible only by this RFC cannot displace an overload that already accepted the source program.

```fsharp
type Stable =
    static member M(value: int64) = "existing"
    static member M(value: byte) = "new"

Stable.M(1) // retains the existing int64 overload
```

The same rule preserves an existing list/sequence overload in preference to a newly available span or builder overload.

### Phase B: contextual literals

Phase B runs only if Phase A has no applicable candidate.

Generic inference is first performed using explicit type arguments and non-contextual arguments. A candidate whose relevant target type remains unresolved is not made applicable by the literal.

Each remaining candidate is checked with contextual literals enabled. Existing F# better-member rules continue to apply. If they do not determine a winner, the following comparisons are used.

#### Numeric literal conversions

A conversion that retains the literal's existing default type is better than one that contextually selects a different numeric type.

Between two non-default contextual numeric targets, existing better-conversion-target relationships are used if one target is unambiguously better. Otherwise the result is ambiguous.

```fsharp
type NewOnly =
    static member M(value: byte) = "byte"
    static member M(value: uint16) = "uint16"

NewOnly.M(1)   // ambiguous
NewOnly.M(1uy) // explicit
```

#### Collection conversions

Collection conversions use the C# 13 "better collection conversion from expression" model:

1. compare conversions of every expression or spread element to the two element types;
2. one element-conversion set is better only when it is no worse for every element and better for at least one;
3. when element types are identical:
   - `ReadOnlySpan<T>` is better than `Span<T>`;
   - a span target is better than an array or listed array-interface target;
4. between non-span collection targets, an unambiguous implicit conversion from one target to the other can make the former better;
5. otherwise neither conversion is better and the call is ambiguous.

This avoids declaration-order tie breaking and makes unrelated targets such as `HashSet<T>` and `List<T>` ambiguous unless element or target conversions establish a winner.

Diagnostics must show the target inferred for each candidate and identify the contextual argument responsible for ambiguity.

# Changes to the F# spec

## Expressions and type checking

Add a contextual-literal checking mode with two entry points:

- a CLI call argument whose candidate parameter type is fully known;
- an expression checked under an explicit type annotation.

The mode may propagate only through the explicitly listed result-type-preserving expression forms. It is not represented as a type constraint and is never generalized.

Define contextual numeric conversion and contextual collection conversion as last-resort conversions after current-language checking.

## Numeric literals

Extend literal checking so an unsuffixed numeric token can be parsed directly for one of the intrinsic target types in the target table. Reuse existing suffixed-literal representation, range, rounding, units-of-measure, and constant-generation rules.

No lexical grammar change is required.

## Collection expressions

Define the eligible element/spread subset of bracket syntax and the valid target categories. Specify target element checking, builder validation, evaluation order, and lowering equivalence.

No new source token is required.

## Method application resolution

Add Phase A/Phase B candidate checking before the existing better-function-member comparison. Record contextual conversion kind, target type, element type, and builder method in typed syntax and FSharp.Compiler.Service data.

## Constraint solving

No changes are made to SRTP constraint syntax, type generalization, the value restriction, or inferred signatures.

# Drawbacks

## One syntax can denote multiple representations

Within an eligible context, `[1; 2; 3]` can construct a list, array, span, interface implementation, or builder-backed collection. Likewise, `1` can be represented as a non-default intrinsic type. Readers may need signature help when the target is not written locally.

## Method calls and F# functions are asymmetric

The first version follows the approved method-call scope. A CLI method can supply context directly, while a curried F# function requires an explicit annotation. This is learnable but not fully orthogonal.

## Refactoring may require an annotation

Extracting a contextual literal into an unannotated binding causes normal defaulting:

```fsharp
Api.M([1; 2; 3])

let values = [1; 2; 3]
Api.M(values)
```

Tooling should preserve the target with an annotation during extraction.

## Compiler and tooling complexity

Candidate checking, constant parsing, collection lowering, span safety, diagnostics, semantic classification, and refactorings all gain new cases. The two-phase compatibility rule limits semantic risk but adds implementation work.

## Target-dependent performance

A concise expression can still allocate, copy, enumerate, or run user builder code. The target signature and tooling, rather than syntax alone, communicate those costs.

# Alternatives

## Keep suffixes and factories only

This is maximally explicit but retains avoidable ceremony at .NET API boundaries and can force an intermediate representation.

## Use only `op_Implicit`

`op_Implicit` from the default source type cannot provide direct target-specific parsing for every narrowing literal, and string/list-based conversions may allocate before conversion. A general multi-step user-conversion search also introduces ambiguity and side-effect questions outside this RFC.

## Make literals generically constrained

The original RFC proposed value, range, tuple, and collection SRTP constraints. The review rejected those changes because of their inference, tooling, and ecosystem impact.

## Permit backwards inference

Allowing later uses, pipelines, or returned values to change an earlier literal would create action-at-a-distance and change inferred signatures. This RFC intentionally does not do so.

## Add a new collection syntax

A new syntax would avoid overloading brackets but would add another collection notation and would not let existing idiomatic F# brackets adapt naturally to a concrete API target.

## Prefer every new span overload

Silently replacing an existing overload winner can change behavior as well as performance. Phase A preserves existing programs; callers can select a span explicitly when needed.

## Support only arrays and spans

That would cover many APIs but omit immutable and domain-specific collections that use the standard .NET collection-builder convention.

# Prior art

- C# implicit constant-expression conversions for representable numeric constants.
- C# target-typed collection expressions, including arrays, spans, collection initializers, interfaces, and `CollectionBuilderAttribute`.
- C# 13 better-conversion rules for collection-expression overload resolution.
- F# FS-1093 type-directed conversions, which establish a last-resort conversion model and warning precedent.
- Existing F# expected-type propagation for lambdas, object expressions, and other explicitly contextual constructs.

# Compatibility

## Source compatibility

Unannotated literals retain their current defaults.

The Phase A rule guarantees that a call which was valid before this RFC retains its pre-RFC applicable candidate set and winner. Contextual conversions are considered only for a call that otherwise has no applicable candidate.

Previously invalid annotated or call-site forms may become valid.

Adding an overload can change a previously Phase-B call or make it ambiguous, as with other overload-resolution features. Explicit suffixes or annotations provide a stable disambiguation.

## Binary compatibility

Existing inferred public signatures and selected members are unchanged. Contextual literals elaborate to existing CLI constants, arrays, spans, constructor/Add calls, or builder calls; no new runtime representation is introduced.

## Older compilers

Older compilers reject source that relies on the new conversion. Assemblies produced by a new compiler expose ordinary CLI types and instructions and remain consumable by older compilers subject to the target framework used by the referenced APIs.

## Language version

The implementation should initially be gated by the corresponding preview language version because the feature changes type checking and overload candidate applicability despite adding no new parser tokens.

# Interop

This RFC is primarily an interop feature.

- Numeric arguments can directly target intrinsic CLI numeric parameter types.
- Bracket expressions can target arrays, spans, collection-initializer types, standard collection interfaces, and `CollectionBuilderAttribute` types.
- Builders and `Add` methods may be authored in C#, F#, or another CLI language.
- No F#-specific metadata is required.

The collection-builder shape deliberately matches .NET/C# so library authors do not need a separate F# construction protocol.

# Pragmatics

## Diagnostics

Diagnostics should distinguish:

- target type not fully known;
- numeric value outside the target range;
- unsupported numeric target or literal form;
- element or spread iteration type incompatible with the target element type;
- invalid or ambiguous collection-builder metadata;
- multiple incomparable Phase-B overloads;
- span/byref-like escape violations.

A range error should identify both the literal and target, for example:

```text
The literal '256' cannot be represented as System.Byte.
The valid range is 0 through 255.
```

An overload diagnostic should list each candidate's contextual target rather than reporting only a generic method-resolution failure.

## Tooling

FSharp.Compiler.Service should expose:

- whether contextual literal checking was used;
- the selected target and element types;
- the selected builder, constructor, and `Add` method where applicable;
- the resulting overload candidate.

Hover should show the contextual representation. Go to definition from a builder-backed expression should navigate to the builder method. Extract-expression refactorings should insert the annotation needed to preserve semantics.

An optional analyzer or warning, disabled by default, may flag an unannotated call-site literal that receives a non-default numeric or non-list collection target.

## Performance

Numeric literals are compile-time constants with no added runtime conversion.

Collection lowering may remove an intermediate F# list and enable static-data, span, capacity, or builder optimizations. No allocation strategy is guaranteed, and user-defined builders or `Add` methods retain their normal observable behavior.

## Scaling

Phase B runs only when ordinary candidate checking finds no applicable member. Implementations should cache per-candidate literal conversion results and builder validation.

Collection materialization scales linearly with the number of produced elements, subject to the behavior of spread enumerators and target builders.

## Culture-aware formatting/parsing

Numeric source syntax remains culture-invariant and uses the existing F# lexical decimal point, exponent, and digit rules. Collection construction performs no culture-sensitive conversion.

# Unresolved questions

- Should a later RFC extend Level 1 to direct application of non-generic F# functions with fully known parameter types?
- Should ranges and the wider F# sequence-expression grammar become contextual collection expressions?
- Should nullable lifting match the complete C# collection-expression conversion set?
- Which additional numeric targets, if any, should be added after implementation experience (`Half`, `Int128`, `UInt128`, or a constrained generic-math protocol)?
- Should FSharp.Core collection types acquire `CollectionBuilderAttribute` metadata where target-framework compatibility permits it?
- Should the optional call-site warning ship as a compiler warning or only as an analyzer?
