# RFC: Type References

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Namespace support](Namespacing.md)
- [RFC: Operation Types](OperationTypes.md)
- [RFC: Allow directives on directives](DirectivesOnDirectives.md)
- [RFC: Introspectable directives](IntrospectableDirectives.md)

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
supported by the schema. Consider the following schema:

```GraphQL
interface Pet { ... }
interface LandPet implements Pet { ... }

type Dog implements LandPet { ... }
type Cat implements LandPet { ... }
type Bird implements Pet { ... }
type Fish implements Pet { ... }

extend type Query {
    findPets(types: [TypeRef<Pet>!]): Pet
}
```

... and the following query:
```GraphQL
query {
    findPets(types: [LandPet, Bird]) { ...}
}
```

... to be completed.