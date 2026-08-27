# F# RFC FS-1346 - Copy-and-update expressions for C# records and structs

The design suggestion [Support F# record syntaxes for C# defined records](https://github.com/fsharp/fslang-suggestions/issues/1138) has been marked "approved in principle".

This RFC is the focused successor to section **t** of [the original omnibus RFC](https://github.com/fsharp/fslang-design/pull/800), as requested by [the design review](https://github.com/fsharp/fslang-design/pull/800#issuecomment-3852354596). It covers copy-and-update only; construction and field-name-driven type inference are outside scope.

- [x] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/1138)
- [x] Approved in principle
- [ ] Implementation
- [ ] [Discussion](https://github.com/fsharp/fslang-design/discussions/FILL-ME-IN)

# Summary

Extend F#'s existing copy-and-update syntax:

```fsharp
{ receiver with Member1 = value1; Member2 = value2 }
```

to values whose independently known static type is:

1. a C#-compatible record class identified by its compiler-generated clone method; or
2. a non-byref-like CLI struct, including C# record structs.

Given C# declarations:

```csharp
public record Person(string Name, int Age);
public readonly record struct Point(int X, int Y);
```

F# can write:

```fsharp
let birthday (person: Person) =
    { person with Age = person.Age + 1 }

let moveX (point: Point) =
    { point with X = 10 }
```

For a record class, the compiler invokes the record's virtual clone operation and then assigns the listed accessible fields or properties on the clone. For a struct, it copies the value and applies assignments to the copy. The original value is not directly assigned to.

Existing F# record and anonymous-record updates retain their current type inference and lowering and are always preferred when applicable.

The new branch does not infer a receiver type from member names:

```fsharp
let rename (person: Person) =
    { person with Name = "Ada" } // valid

let renameUnknown person =
    { person with Name = "Ada" } // no new generic C#-record inference
```

# Motivation

C# records are designed around non-destructive mutation through `with` expressions. F# can construct and read these types, and it understands their init-only properties, but it cannot currently express the record's intended copy-and-update operation directly.

Without language support, callers must use awkward or unsafe alternatives:

- reconstructing the value through a constructor, which may omit non-positional state or lose the runtime derived type;
- reflection or generated helpers;
- hand-written copy methods for every foreign record;
- mutating a property in place, which is impossible for init-only members and changes semantics for mutable records;
- calling the compiler-reserved `<Clone>$` method, which is not a normal source-level API.

Reusing F# record-update syntax is natural because both features mean “copy this value and replace selected members”. It also preserves C# record inheritance semantics: a virtual clone of a base-typed value retains its runtime derived record type.

C# record structs do not carry a reliable record marker in CLI metadata. C# therefore defines `with` for structs generally by copying the value and assigning members. Supporting non-byref-like structs gives F# correct record-struct interop without inventing a fragile metadata heuristic.

# Detailed design

## Syntax

No grammar is added. The existing record-update expression is reused:

```fsgrammar
{ expr with field-initializers }
```

The result is still an expression and may appear anywhere an expression is permitted.

The assignment list must contain at least one member under the existing grammar, and duplicate member names remain an error.

## Choosing the update interpretation

The compiler checks an update expression in this order.

### 1. Existing F# record and anonymous-record update

Current F# behavior is attempted first, unchanged. This includes:

- F# record-field label inference;
- struct and reference F# records;
- anonymous records;
- existing accessibility, field completeness, and copy/update lowering rules.

A program that currently compiles as an F# record update continues to use that interpretation even if the compiled type also happens to contain members resembling a C# record clone.

### 2. Nominal copy-and-update

If the existing interpretation does not apply, the receiver expression is checked independently of the update labels.

The new interpretation is available only when the resulting static receiver type is fully known and is either:

- an eligible record class; or
- an eligible struct.

The member names do not create constraints that search for a nominal type. In particular, a bare type parameter or unresolved inference variable is not made “record-like” by the update expression.

A constructed generic type is permitted when its type arguments are already known:

```fsharp
let replace (box: CSharpBox<int>) =
    { box with Value = 42 }
```

The update preserves the same constructed type and cannot change generic arguments.

## Eligible record classes

A reference type is eligible when the compiler finds exactly one valid C# record clone method using the Roslyn metadata convention.

The method must:

- have compiled name `<Clone>$`;
- be public;
- be an instance method;
- have no parameters;
- have generic arity zero;
- be the only candidate satisfying these shape requirements;
- be virtual, override, or abstract, unless the containing type is sealed;
- return a type from which the containing record type is equal to or derived.

The final rule accommodates record inheritance and assemblies targeting runtimes without covariant-return support, where an override can retain a base-record return type.

If no unique valid method exists, the reference type is not eligible. The compiler does not fall back to a public copy constructor, `ICloneable`, a user method named `Clone`, serialization, or reflection.

The metadata convention, rather than the source language that produced it, is authoritative. A type from any CLI language that deliberately exposes the same valid shape receives the same behavior, just as C# would recognize the shape.

## Eligible structs

Any statically known CLI value type is eligible except:

- F# record/anonymous-record values already handled by the existing branch;
- byref-like (`ref struct`) types in the initial implementation;
- nullable wrappers whose underlying value is not first made explicit;
- a bare generic type parameter, even when constrained to `struct`.

C# record structs are included. Ordinary mutable structs are also included because record structs have no reliable distinguishing marker in metadata and C# `with` expressions use copy-and-assign semantics for structs generally.

Readonly structs are eligible when the requested updates target accessible setters that are valid during initialization, typically init-only properties. Readonly fields remain non-assignable.

Byref-like structs are excluded initially because the generated mutable temporary and expression result must participate in F#'s byref-safety and escape analysis. They can be considered in a focused follow-up.

## Eligible update members

Each update label is resolved against the already-known static receiver type using ordinary nominal member lookup.

A valid target is an accessible instance member that is one of:

- a non-readonly field; or
- a non-indexed property with an accessible `set` or `init` accessor.

Static members, events, methods, indexers, get-only properties, readonly fields, and inaccessible setters are rejected.

Inherited fields and properties participate according to ordinary accessibility and hiding rules. No extension-property convention is introduced.

The existing `long-ident = expr` grammar is retained. If a label is qualified, its qualifier must resolve consistently with ordinary F# member/declaring-type qualification; qualification does not enable field-label type inference for this branch.

Every member may appear at most once in an update list. The right-hand expression is checked against the member's value type using existing F# conversion and expected-type rules.

## Record-class lowering

For a receiver with static record-class type `R`, the expression is semantically equivalent to:

```fsharp
let receiverTemp: R = receiver
let cloneResult = receiverTemp.<Clone>$()
let mutable resultTemp: R =
    // identity/reference conversion, or compiler-generated checked cast
    // when an older covariant-return shape returns a base record type
    convertCloneResult cloneResult

resultTemp.Member1 <- value1
resultTemp.Member2 <- value2
resultTemp
```

The compiler-reserved method is invoked directly from metadata; users do not need source syntax for its name.

More precisely:

1. evaluate the receiver exactly once;
2. invoke the selected clone method exactly once using ordinary virtual dispatch;
3. convert the clone result to the static receiver type, including a runtime cast when required by the valid metadata shape;
4. evaluate each right-hand expression and perform its assignment in lexical order;
5. return the updated clone with the static receiver type.

Virtual dispatch is important:

```fsharp
let updateBase (value: BaseRecord) =
    { value with BaseProperty = 1 }
```

If `value` contains a derived record, its overriding clone operation produces a derived instance, as in C#.

The compiler trusts the valid record-clone contract. A deliberately malformed metadata type whose clone returns the same object or performs unusual side effects has those ordinary method semantics.

## Struct lowering

For a receiver with static struct type `S`, the expression is semantically equivalent to:

```fsharp
let mutable resultTemp: S = receiver
resultTemp.Member1 <- value1
resultTemp.Member2 <- value2
resultTemp
```

The receiver is evaluated and copied once. Assignments and right-hand expressions occur in lexical order. The resulting value is the modified copy.

This may copy a large struct, exactly as a C# `with` expression does. The compiler may optimize copies when observable behavior is preserved.

## Init-only setters

F# already recognizes the `IsExternalInit` modifier and normally restricts init-only setters to object initialization.

Assignments generated by this RFC are considered part of the initialization of the fresh clone or struct copy. Therefore an accessible init-only property is valid:

```fsharp
let renamed = { person with Name = "Ada" }
```

This permission is narrowly scoped to the compiler-generated result temporary. It does not allow an init-only property to be assigned on an arbitrary existing value:

```fsharp
person.Name <- "Ada" // remains an error
```

## Required members

Required-member checking is not rerun for a copy-and-update expression. The receiver is already a constructed value, and the clone/copy preserves all existing state. Only members named by the update are assigned.

A required member may be updated when it is otherwise writable, but omitted required members do not need to be listed again.

## Evaluation order and exceptions

Evaluation order is:

1. receiver;
2. clone/copy;
3. first right-hand expression;
4. first assignment;
5. second right-hand expression;
6. second assignment;
7. and so on.

Every right-hand expression is evaluated exactly once.

Property getters used inside a right-hand expression, property setters, field assignments, conversion operators already permitted by F#, and the clone method retain their ordinary side effects and exception behavior.

If an operation throws, the exception propagates. Earlier assignments to the temporary may already have occurred, but the result value is not returned. Effects performed by setters or right-hand expressions are not rolled back.

## Null behavior

For a record class, a null receiver reaches an ordinary instance clone call and throws `NullReferenceException`. Existing nullable-reference analysis should diagnose a possibly-null receiver according to the current nullness mode.

The feature does not interpret null as a default record, skip the update, or return null.

## Result type

The result has exactly the static receiver type, including all generic arguments and nullness information resulting from existing rules.

The update cannot change a generic parameter, select a derived static type from member labels, or infer a new nominal type.

## Restrictions

This RFC does not add:

- C# record construction using `{ Member = value }`;
- positional-record constructor inference;
- nominal type inference from field/property names;
- generic “record” or “cloneable” constraints;
- updates through an interface or `obj` based on runtime type;
- updates to arbitrary reference types with a copy constructor;
- property/field patterns or `Deconstruct` support;
- conversion between F# and C# record types;
- update of indexers or collection contents;
- compound assignments in update initializers;
- special required-member revalidation;
- byref-like struct support in the first implementation.

# Changes to the F# spec

## Record expressions

Retain the existing grammar and divide copy-and-update checking into:

1. the existing F# record/anonymous-record interpretation; and
2. nominal copy-and-update for an independently known eligible record class or struct.

State that only the first interpretation may use record-field labels to infer the receiver type.

## Record-class recognition

Define the `<Clone>$` metadata predicate and uniqueness, accessibility, virtual/sealed, parameter, arity, and return-type requirements listed above.

## Member initializers

Define eligible instance fields and properties, duplicate detection, expected-type checking, accessibility, init-only permission, and lexical evaluation order.

## Lowering

Specify clone/convert/assign/return semantics for record classes and copy/assign/return semantics for structs. Preserve virtual dispatch, receiver evaluation count, assignment order, exceptions, null behavior, and result type.

## Type inference

No nominal type variable is inferred from update labels in the new branch. Bare type parameters and unresolved receiver types are not eligible. Existing F# record-field inference remains unchanged and takes precedence.

# Drawbacks

## The same syntax has two metadata models

An F# record update copies compiler-known record fields, while a C# record update invokes user-visible runtime behavior through a hidden clone method and property setters. Tooling must make the selected lowering clear.

## Clone and setter code may have effects

C# records normally synthesize predictable cloning, but metadata can provide custom copy constructors, clone overrides, and setters. An apparently declarative update can allocate, log, mutate external state, or throw.

## Struct support is broader than record structs

Because CLI metadata does not reliably distinguish C# record structs, the design allows eligible structs generally. This matches C# but means the F# syntax is not exclusively a “record” operation in the nominal branch.

## Large structs may be expensive to copy

Copy-and-update necessarily starts from a value copy. Repeated updates of large structs can be more expensive than direct mutation or a purpose-built API.

## The receiver type must already be known

The feature cannot define a generic function merely from member labels. An annotation is often necessary at abstraction boundaries, but this is the price of avoiding open-ended field-name inference across all nominal types.

## Byref-like structs are deferred

The initial scope omits `ref struct` values even where a local copy could theoretically be safe.

# Alternatives

## Require a new syntax for C# records

A form such as `receiver with { Member = value }` could make the metadata model explicit and resemble C#. It would add syntax for an operation that already has a direct F# semantic analogue.

## Call a public copy constructor

C# record inheritance semantics depend on a virtual clone operation. A copy constructor selected from the static type can slice a derived record and is not uniformly accessible.

## Use `ICloneable`

`ICloneable.Clone` returns `obj`, has underspecified shallow/deep semantics, is not synthesized by C# records, and does not enable init-only assignments.

## Require a user-authored helper or lens

Helpers are explicit and can support arbitrary types, but requiring adapters for every C# record defeats basic interop and duplicates compiler-known metadata semantics.

## Infer nominal types from member names

Searching all visible nominal types for matching properties would be expensive, ambiguous, sensitive to imports, and a major expansion of F# field-label inference. The suggestion discussion and design review explicitly favor a known receiver type.

## Support record classes but not structs

That would leave C# record structs unsupported because they have no class-style clone marker. Detecting them heuristically from synthesized equality or printing members would be fragile. General struct copy semantics are simple and match C#.

## Support every reference type by memberwise copy

The CLI has no safe general memberwise-copy protocol accessible to F#. Reflection or `MemberwiseClone` would violate accessibility and object invariants. The `<Clone>$` shape is the explicit C# record contract.

# Prior art

- F# record and anonymous-record copy-and-update expressions.
- C# record-class `with` expressions: virtual clone followed by lexical-order member initialization.
- C# struct `with` expressions: value copy followed by lexical-order member initialization.
- Roslyn's `<Clone>$` metadata recognition rules for record classes.
- F# support for consuming init-only and required properties under FS-1127.
- Functional lenses and generated `With...` methods provide library-level alternatives for arbitrary object models.

# Compatibility

## Source compatibility

Existing valid F# record and anonymous-record updates retain first priority, current inference, selected fields, and lowering.

The feature makes previously invalid update expressions over known C# record classes and eligible structs valid. It does not reinterpret an existing valid nominal expression.

A type can become newly eligible if a later assembly version adds a valid `<Clone>$` method or changes member writability. That affects only source relying on the nominal update branch and follows ordinary library-versioning behavior.

## Binary compatibility

No new metadata is emitted for the expression. Compiled code contains ordinary clone calls, casts, struct copies, field stores, and property-setter calls.

## Older compilers

Older F# compilers reject source using record-update syntax with a non-F# record receiver. They can consume assemblies produced from the new syntax because the emitted IL uses existing CLI members and types.

## FSharp.Core

No FSharp.Core addition is required.

## Language version

The nominal branch should initially be gated by the corresponding preview language version. Existing F# record behavior is unchanged under all versions.

# Interop

The feature is defined by CLI metadata rather than C# syntax and therefore supports:

- C# record classes, including inheritance and covariant-return variations;
- C# record structs and readonly record structs;
- ordinary CLI structs with writable or init-only members;
- equivalent metadata emitted by another .NET language or generator.

Public APIs and result values keep their original nominal types. No wrapper, reflection metadata, or F# adapter is introduced.

# Pragmatics

## Diagnostics

Diagnostics should distinguish:

- receiver type not independently known;
- reference type is not a recognized record class;
- missing, invalid, or ambiguous `<Clone>$` metadata;
- unknown member on the static receiver type;
- static, indexed, readonly, get-only, or inaccessible member;
- duplicate member initializer;
- right-hand expression incompatible with member type;
- byref-like struct excluded by the initial scope;
- nullable receiver warning followed by ordinary runtime semantics.

For a class that nearly matches the record shape, the diagnostic should explain which clone requirement failed rather than report only “not an F# record”.

## Tooling

FSharp.Compiler.Service should represent nominal copy-and-update distinctly from F# record update so tools do not assume F# record-field symbols.

- Completion after `with` should list accessible writable fields and properties.
- Hover should show the static receiver/result type and whether cloning or struct copying is used.
- Go to definition on the update expression should navigate to `<Clone>$` where the UI can expose compiler-generated members; member labels navigate to their field/property or setter.
- Find references and call hierarchy should account for clone and setter calls.
- Refactorings must not remove the receiver annotation when it is what makes the nominal type known.

## Performance

Record-class updates allocate according to the record's clone implementation and then perform one assignment per listed member.

Struct updates copy the receiver and then assign. The compiler may elide redundant temporaries or copies when doing so preserves side effects, aliasing, exception order, and result identity.

No reflection, dictionary, expression tree, or general-purpose cloning helper is required.

## Scaling

Record-shape recognition is a bounded lookup for one reserved method name. Member resolution scales with the number of initializers and ordinary nominal lookup. No global search by field name occurs.

## Culture-aware formatting/parsing

N/A. The feature performs no culture-sensitive parsing or formatting; right-hand expressions retain their ordinary semantics.

# Unresolved questions

- Should byref-like structs be enabled later when escape analysis proves the generated temporary safe?
- Should a future generic constraint permit copy-and-update on a type parameter with a statically known record clone/member shape?
- Should tooling surface the compiler-generated `<Clone>$` name or present it as a language-neutral “record clone” operation?
- Should a future RFC add construction syntax for C# records, or is ordinary constructor/object-initializer syntax sufficient?
