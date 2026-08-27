# F# RFC FS-1344 - `Deconstruct` method support in tuple patterns

The design suggestion [Support C#-style Deconstruct method based pattern matching](https://github.com/fsharp/fslang-suggestions/issues/751) has been marked "approved in principle".

This RFC is the focused successor to section **h** of [the original omnibus RFC](https://github.com/fsharp/fslang-design/pull/800), as requested by [the design review](https://github.com/fsharp/fslang-design/pull/800#issuecomment-3852354596).

- [x] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/751)
- [x] Approved in principle
- [ ] Implementation
- [ ] [Discussion](https://github.com/fsharp/fslang-design/discussions/FILL-ME-IN)

# Summary

Allow an ordinary F# tuple pattern to consume a value whose already-known, non-tuple type exposes a C#-compatible `Deconstruct` method.

```fsharp
type Person(name: string, age: int) =
    member _.Deconstruct(nameResult: string outref, ageResult: int outref) =
        nameResult <- name
        ageResult <- age

let name, age = Person("Ada", 36)
```

The selected method has the CLI shape:

```csharp
void Deconstruct(out T1 item1, out T2 item2, ...)
```

Accessible extension methods are also supported, enabling idiomatic consumption of types authored in C# or extended by another library.

Existing tuple inference remains the default:

```fsharp
let first, second = value
```

If the type of `value` is not yet known, the pattern continues to infer a normal F# tuple. `Deconstruct` lookup is used only after the pattern input is independently known to be a concrete non-tuple type. This prevents the feature from changing the inferred signatures of existing code.

The RFC adds no new pattern syntax, no SRTP constraint, and no field/property pattern facility.

# Motivation

C# libraries increasingly expose positional decomposition through `Deconstruct` methods. C# records synthesize such methods, and ordinary classes and extension libraries can provide them explicitly.

F# can call these methods manually:

```fsharp
let mutable name = Unchecked.defaultof<string>
let mutable age = 0
person.Deconstruct(&name, &age)
```

but this loses the main benefit of the convention: concise, nested pattern decomposition.

F# users currently need to add a bespoke active pattern for every consumed type:

```fsharp
let (|PersonParts|) (person: Person) =
    // call Deconstruct and return a tuple
    ...

match person with
| PersonParts (name, age) -> ...
```

That approach is useful when an F# API author wants custom matching semantics, but it is unnecessary ceremony when a .NET type has already declared the ecosystem-standard decomposition contract.

Reusing tuple patterns provides:

- direct interoperation with C# records and `Deconstruct`-based APIs;
- nested F# patterns over the extracted values;
- support for both instance and extension-based deconstruction;
- no new syntax to learn;
- preservation of current F# tuple inference when the input type is not known.

# Detailed design

## Eligible pattern

The fallback applies to ordinary reference-tuple pattern syntax containing at least two component patterns:

```fsharp
p1, p2
(p1, p2, p3)
```

The explicit struct-tuple form remains exclusively a struct-tuple pattern and never performs `Deconstruct` lookup:

```fsharp
struct (p1, p2)
```

There is no zero- or one-component tuple pattern in F#, so `Deconstruct()` and `Deconstruct(out T)` are outside this RFC.

Named `Deconstruct` parameters do not add named subpattern syntax. Components are matched positionally.

## Selection between tuple and `Deconstruct`

When checking a tuple pattern of arity `N`, the compiler applies these cases in order.

### 1. The input type is not yet known

If the pattern input contains an unresolved inference variable at the point where its shape must be established, current F# behavior is retained: the input is constrained to an F# reference tuple of arity `N`.

```fsharp
let pairFunction (x, y) = x, y
// remains: 'a * 'b -> 'a * 'b
```

The compiler does not create a deferred “tuple or Deconstruct” constraint and does not revisit this decision based on later uses.

### 2. The input is an F# tuple

If the independently known input type is a reference tuple of arity `N`, current tuple matching is used.

If it is a reference tuple of another arity, the existing arity diagnostic is produced.

An explicit `struct (...)` pattern continues to handle struct tuples. An ordinary tuple pattern does not silently switch an F# reference tuple to a struct tuple as part of this RFC.

### 3. The input is a known non-tuple type

The compiler attempts `Deconstruct` lookup for the known static input type and arity `N`.

If a valid method is selected, its output values become the inputs of the corresponding component patterns.

If no method is selected, the compiler reports the existing tuple-type mismatch augmented with information about the failed `Deconstruct` lookup.

Examples where the input type is already known include:

```fsharp
// The right-hand side fixes the input type.
let name, age = getPerson()

// The match input fixes the input type.
match person with
| name, age -> name, age

// An explicit pattern annotation fixes the input type.
let describe ((name, age): Person) =
    $"{name}: {age}"

// The sequence element type fixes the input type.
for name, age in people do
    printfn "%s" name
```

An expected function type may also fix a lambda parameter before the pattern is checked:

```fsharp
let consume: Person -> string =
    fun (name, age) -> $"{name}: {age}"
```

## Valid `Deconstruct` methods

A candidate method must satisfy all of these requirements:

- its compiled name is `Deconstruct`;
- it is an accessible instance method, or an accessible extension method applicable to the input type;
- it returns CLI `void` (the normal compiled form of an F# member returning `unit`);
- apart from the extension receiver, it has exactly `N` parameters;
- every one of those `N` parameters is a CLI `out` parameter;
- it is not a property accessor or other special-name member masquerading under source syntax.

Inherited instance members participate through ordinary member lookup. Static methods that are not extension methods do not participate.

For example, an F# extension can provide decomposition for an existing type:

```fsharp
open System.Runtime.CompilerServices

[<Extension>]
type KeyValuePairDeconstruction =
    [<Extension>]
    static member Deconstruct
        (pair: System.Collections.Generic.KeyValuePair<'K, 'V>,
         key: 'K outref,
         value: 'V outref) =
        key <- pair.Key
        value <- pair.Value
```

## Overload resolution

Method selection follows ordinary F#/.NET member and extension-member lookup, adapted to synthetic output arguments:

1. applicable instance members are considered before extension members, as in ordinary member lookup;
2. exactly `N` fresh output locations are supplied to candidate checking;
3. generic type arguments may be inferred from the receiver, declaring type, and ordinary constraints already fixed independently;
4. component subpatterns and their type annotations do **not** supply input evidence for choosing an overload;
5. after a method is selected, each component pattern is checked against the corresponding output type.

The output locations intentionally begin without user-supplied types. Consequently, two otherwise applicable `Deconstruct` overloads with the same arity but different output types are ambiguous:

```csharp
void Deconstruct(out int x, out int y);
void Deconstruct(out string x, out string y);
```

A nested annotation does not choose between them:

```fsharp
let (x: int), y = value // still ambiguous at Deconstruct selection
```

This matches the versioning intent of the .NET convention: arity selects a decomposition shape, while output types are the result of that selected shape rather than call-site arguments.

A generic method whose type parameters occur only in output positions is not applicable because those parameters cannot be inferred independently.

If ordinary lookup finds multiple incomparable extension methods, the pattern is ambiguous and the diagnostic lists them.

## Pattern checking after extraction

Once a method is selected, each out-parameter type becomes the input type of the corresponding nested pattern:

```fsharp
match person with
| ("Ada", age) when age >= 18 -> Adult age
| (_, age) -> Minor age
```

All existing pattern forms may be nested, including constants, union cases, records, arrays, active patterns, type tests, `as` patterns, and further tuple/`Deconstruct` patterns.

The `Deconstruct` call itself is an unconditional extraction operation. Whether the overall pattern succeeds is determined by the nested patterns and guard.

A tuple pattern backed by `Deconstruct` is treated as exhaustive to the same extent as an ordinary tuple pattern with the same component patterns. The compiler does not assume that the method is pure or non-throwing.

## Evaluation and runtime behavior

For each occurrence of a `Deconstruct`-backed pattern that is tested:

1. the pattern input is evaluated or loaded exactly once;
2. the selected method is invoked once;
3. each out value is stored in a fresh compiler-generated location;
4. component patterns are tested left-to-right using those stored values, subject to the existing freedom of the F# pattern-match compiler where no observable difference is introduced.

The compiler may share one extraction among tests within the same compiled decision path, but must not duplicate an effectful `Deconstruct` invocation for one logical test of the pattern.

Different match clauses containing separate `Deconstruct` patterns may invoke the method separately, just as separate active-pattern occurrences may execute separately.

An instance method uses ordinary virtual dispatch. If a null value reaches an instance-method-backed pattern, invocation has ordinary null instance-call behavior. An extension method receives the value as its ordinary first argument, including `null` when permitted by the static type. Existing nullness analysis should warn where appropriate.

Exceptions thrown by the method propagate normally; they do not mean “pattern did not match”.

## Restrictions

This RFC does not add:

- deferred or backwards inference of a `Deconstruct` constraint;
- statically resolved `Deconstruct` constraints;
- dynamic runtime lookup on `obj`;
- `ITuple` fallback;
- zero- or one-output decomposition syntax;
- named positional subpatterns based on out-parameter names;
- implicit matching of fields or properties not returned by the method;
- tuple construction conversions in the opposite direction;
- special handling for `op_Implicit` to a tuple.

Active patterns remain the mechanism for user-defined partial matching, validation, caching, or decomposition semantics that do not fit the CLI convention.

# Changes to the F# spec

## Tuple patterns

Extend the tuple-pattern checking algorithm with the ordered cases described above:

1. unresolved input shape: infer a reference tuple, as today;
2. known reference-tuple input: use current tuple semantics;
3. known non-tuple input: attempt a valid `Deconstruct` method of matching arity.

Clarify that the `struct (...)` pattern is not eligible for this fallback.

## Member lookup

Define `Deconstruct` candidate eligibility and synthetic out-argument overload resolution. Instance and extension member lookup use the existing accessibility, generic inference, constraint, inheritance, and ambiguity rules except that component subpatterns do not participate in candidate selection.

## Pattern compilation

Add a decomposition test/extraction node, or lower to an equivalent method call plus fresh values, while preserving receiver evaluation, invocation count, virtual dispatch, exception behavior, and nested-pattern semantics.

## Inference

State explicitly that the feature creates no new type constraint. An unresolved tuple pattern continues to constrain its input to an F# reference tuple immediately, preserving current generalization and inferred signatures.

# Drawbacks

## Type-directed behavior is less explicit than an active pattern

The same `(p1, p2)` syntax can mean tuple projection or a method invocation depending on the already-known input type. Tooling must make the selected interpretation visible.

## `Deconstruct` methods may have side effects

A pattern can invoke user code that allocates, mutates state, logs, blocks, or throws. Active patterns already permit effectful matching, but ordinary tuple syntax may make the call less obvious.

## Same-arity overloads are fragile

Adding another `Deconstruct` overload with the same arity can make consuming source ambiguous. API authors should distinguish decomposition shapes by arity, which is also the common C# guidance.

## Unknown inputs remain tuples

A generic function cannot infer “anything deconstructable” from `(x, y)`. This is an intentional compatibility boundary, but it means an annotation or other expected type is required in some abstractions.

## No partial match at the extraction layer

`Deconstruct` has no Boolean success result. A type that needs extraction to fail should expose an active pattern or a `Try...` API rather than relying on this feature.

# Alternatives

## Require explicit `Deconstruct(p1, p2)` syntax

An explicit compiler-known pattern would make method invocation visible and avoid overloading tuple syntax. It would add a new special pattern name and be less idiomatic for .NET types designed around positional deconstruction.

## Generate or require active patterns

Active patterns are explicit and fully customizable, but requiring one adapter per foreign type defeats the interoperability value of the established metadata convention.

## Defer tuple inference with a new constraint

A “tuple or deconstructable” constraint could make generic tuple-pattern functions work with arbitrary types. The design review rejected this broader constraint/inference direction because it would alter generalization, diagnostics, tooling, and existing inferred signatures.

## Use `op_Implicit` to a tuple

A conversion operator changes more than decomposition: it constructs and exposes a tuple value throughout expression typing. `Deconstruct` is specifically designed for positional extraction, supports extension methods, and can evolve by adding arities.

## Use field or property names automatically

Property patterns are useful but are a separate design with questions around syntax, accessibility, inheritance, partiality, and exhaustiveness. This RFC implements only the approved `Deconstruct` convention.

## Match `ITuple` dynamically

A runtime `ITuple` fallback would permit decomposition of values whose static type is `obj`, but it would add runtime arity/type tests and weakly typed component values. It is not required for C# `Deconstruct` interop.

# Prior art

- C# deconstruction declarations and positional patterns use accessible instance or extension `Deconstruct` methods with `out` parameters.
- C# records synthesize `Deconstruct` for positional members.
- F# active patterns provide explicit, programmable decomposition and remain the more general facility.
- F# already uses expected input types to interpret several pattern forms while retaining normal tuple inference for unknown inputs.

# Compatibility

## Source compatibility

An ordinary tuple pattern whose input type is unresolved continues to infer exactly the same reference-tuple type as before.

An ordinary tuple pattern over a known tuple continues to use tuple semantics.

The feature makes previously invalid patterns over known non-tuple types valid when a suitable method exists. It does not replace an existing valid interpretation.

A future library version that adds or overloads `Deconstruct` can change the compilation of source that relies on this fallback, including introducing ambiguity. This is analogous to extension-method and overload versioning elsewhere in .NET.

## Binary compatibility

No new metadata is emitted for the pattern. Compiled code contains ordinary calls to existing methods and ordinary locals for the out values.

## Older compilers

Older F# compilers reject source that relies on tuple syntax over a non-tuple input. They can consume assemblies produced from the new syntax because no new runtime representation is introduced.

## Language version

The fallback should initially be gated by the corresponding preview language version. Current tuple-pattern behavior remains available under older language versions.

# Interop

The feature consumes the same CLI method shape used by C# and other .NET languages. It supports:

- C# records with synthesized positional deconstruction;
- hand-written C# instance methods;
- C# or F# extension methods;
- inherited and virtual methods;
- generic declaring types when type arguments are already known.

No FSharp.Core helper, F#-specific attribute, or wrapper allocation is required.

# Pragmatics

## Diagnostics

Diagnostics should distinguish:

- no accessible `Deconstruct` with the requested arity;
- a candidate with a non-void return;
- a candidate containing a non-`out` parameter;
- generic type arguments that cannot be inferred independently;
- same-arity overload ambiguity;
- extension-method ambiguity;
- nested pattern incompatibility after a method was selected.

An error should report both the static input type and requested arity, and list near-miss methods where useful.

## Tooling

FSharp.Compiler.Service should expose the selected method and output types in typed pattern data.

- Hovering the tuple pattern should show whether it is a tuple or `Deconstruct` pattern and display the selected signature.
- Go to definition from the tuple punctuation or a component should navigate to the method when method-backed.
- Find references and call hierarchy may count the generated method invocation.
- Semantic highlighting may give method-backed tuple punctuation a distinct classification if editors choose.
- Refactorings that extract or generalize a pattern should retain an annotation when removing it would revert the input to tuple inference.

## Performance

The feature emits one method call and one temporary per out parameter for each tested pattern occurrence. No intermediate tuple allocation is required.

The compiler may optimize temporaries and share extraction inside one decision path when this preserves observable behavior. It must not assume user `Deconstruct` methods are pure.

## Scaling

Candidate search is bounded by ordinary member and extension lookup on the statically known type. Pattern compilation then scales with the number and nesting depth of component patterns as it does for tuples today.

## Culture-aware formatting/parsing

N/A. The feature performs no culture-sensitive parsing or formatting.

# Unresolved questions

- Should a future explicit pattern form support zero- and one-output `Deconstruct` methods?
- Should out-parameter names later be usable in a separate named positional-pattern proposal?
- Should null instance receivers eventually fail a `match` pattern rather than follow ordinary instance-call behavior, and could that be done without making binding and match contexts inconsistent?
- Should tooling offer a code fix that generates an equivalent active pattern when callers want explicit or partial semantics?
