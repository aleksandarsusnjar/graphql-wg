# RFC: Type References

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Namespace support](Namespacing.md)
- [RFC: Enhanced unions](EnhancedUnions.md)
- [RFC: Anonymous types](AnonymousTypes.md)

**Impacts:**
- [RFC: Input expressions that enable powerful DSLs](InputExpressions.md)

**See also**


## Problem Statement

Sometimes it necessary to pass an entity type as an input argument in GraphQL operations,
such as in:

- Search criteria restricted to one or more (sub)types, when using polymorphic searching.
- Introspection, e.g. `__type(name: "...")`.
- Subscriptions to event relevant to one or more supported (sub)types.
- etc.

It would be useful to have those references be more than raw `String` values and be
supported by the schema. 

## Proposal

1. Introduce an implicit types `InputType` and `OutputType` that behave
   like an `enum` whose values are all fully-qualified input / output types
   of all kinds. Note that these would *not* be enclosed in quotation marks.
   Note that scalars and enums would be present in both.
   See [RFC: Namespace support](Namespacing.md) for what "fully qualified"
   means. 

2. Allow both `InputType` and `OutputType` to be used as either a
   field output type, an input argument type and/or directive argument type.
   Yes, clients should be able to specify ouput types in input arguments.

3. Allow both `InputType` and `OutputType` uses to be "decorated" in a style
   similar to generic in some languages, using "angle brackets" `<`...`>`.
   Witin the brackets is an implicit and [anonymous](AnonymousTypes.md) 
   [union](EnhancedUnions.md) declaration without the need to include the
   `union` keyword - i.e. it is a `|`-separated list of type names that
   are either fully-qualified or locally deterministic within the context.
   See [RFC: Namespace support](Namespacing.md) for details.

4. Initially neither `InputType` nor `OutputType` need to listed in
   `__Schema.types` or appear in each other, though technically they 
   should. This is to avoid probles with huge composed schemas with 
   potentially millions of types. They would, however, need to appear as
   dedicated new `__TypeKind`s and can leverage `__Type.possibleTypes`
   to list the restrictions described in (3). Again, to avoid type 
   explosion in large schemas, these should not be automatically expanded
   to include indirect possible types (interface subtypes, union options).
   For the same reason they should not be listed in `__Type.enumValues`.

### Example schema

```GraphQL
interface Pet { ... }
interface LandPet implements Pet { ... }

type Dog implements LandPet { ... }
type Cat implements LandPet { ... }
type Bird implements Pet { ... }
type Fish implements Pet { ... }

extend type Query {
    findPets(types: [OutputType<Pet>!]): Pet
}
```

### Example query

```GraphQL
query {
    findPets(types: [LandPet, Bird]) { ...}
}
```

