# RFC: Permit interface types in concrete output

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Namespace support](Namespacing.md) (desired but optional)
- [RFC: Operation Types](OperationTypes.md) (desired but optional)

To support scatter-gather aggregation of polymorphic output(s) such that some data sources
can add fields to objects (also) coming from other sources without having to know the exact
type, we cannot require that the concrete type is known.

Consider the following example:

Common definition:

```GraphQL
interface Entity {
  id: ID!
}
interface Animal {
  species: String!
}
interface Pet implements Entity, Animal {
  id: ID!
  species: String!
}
extend type Query { # or extend type Operation
  pet(id: ID!): Pet
  pets: [Pet!]!
}
```

Dog store server:
```GraphQL
type Dog implements Pet {
  id: ID!
  species: String!
  barkLoudness: Int  
}
type Query { # or Operation
  pet(id: ID!): Pet
  pets: [Pet!]!
}

```
Cat store server:
```GraphQL
type Cat implements Pet {
  id: ID!
  species: String!
  whiskerLength: Int  
}
type Query { # or Operation
  pet(id: ID!): Pet
  pets: [Pet!]!
}
```

```
Ownership server:
```GraphQL
extend interface Pet {
  owner: Person
}
type Person {...}
type Query { # or Operation
  pet(id: ID!): Pet
  pets: [Pet!]!
}
```

We want an auto-aggregating server to be able to obtain pets from the dog and cat store servers and then get ownership details from the ownership server, which does not know about the concrete type of the pet, nor it cares about it.

To address this:

1. We introduce the special `graphql.Anonymous` type only to be used as the possible value of `__typename` output field. It cannot be referenced directly within the schema or introspected. This is simply to maintain that `__typename` is a non-null `String!`.
2. (Optional) We introduce the special `graphql.types: {graphql.Type!}!` and/or `__types: {__Type!}!` field as a bag (set) of types that can be directly resolved. It is only nullable to support those servers that disable introspection in production.
3. At least one of the following:
  - We introduce the special `graphql.typeids: {ID!}!` and/or `__typeids: {ID!}!` field that list all applicable `interface` and `type` identifiers known to the source.
  - We introduce the special `graphql.interfaceids: {ID!}!` and/or `__interfaceids: {ID!}!` field that list all applicable `interface` identifiers known to the source.
4. We can also introduce `graphql.typeid: ID!` and/or `__typeid: ID!` as a new version of `__typename` but this is not required.

... and we allow the servers (and server frameworks) to identigy their instances by their interfaces only and the `graphql.Anonymous` type.

## Compatibility Considerations

Legacy clients never encountered and, thus, never requested `__interfaceids`.

New clients will either probe for or notice the the support in the introspectable schema.