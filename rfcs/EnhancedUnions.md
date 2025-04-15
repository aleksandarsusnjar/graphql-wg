# RFC: Enhanced unions

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Safe navigation (shortcut) and refactoring operators](SafeNavigationAndRefactoring.md)
- [RFC: Anonymous types](AnonymousTypes.md)

**Help with:**
- [RFC: In-data errors](InDataErrors.md)
- [RFC: Input Unions](InputUnions.md)
- [RFC: Input expressions that enable powerful DSLs](InputExpressions.md)

## Problem statement

Presently unions are restricted to "combining" output object types and nothing else. 
However, more functionality is needed to implement the following RFC cleanly
and as described:

- [RFC: In-data errors](InDataErrors.md)
- [RFC: Input Unions](InputUnions.md)
- [RFC: Input expressions that enable powerful DSLs](InputExpressions.md)

Spefifically:

- Scalar-only unions may be desired if the outputs can take different forms,
  say different number types or even enums.

- Mixture of scalar and object (compound, concrete or abstract) types, for
  example with a scalar as the normal output but a compound output in unforeseen
  (error) cases.

- To allow set unions when combining possibly multiple unions and other types.

- To allow complete equivalence with `@oneOf` for input unions.


## Background

Reasons why unions cannot reference scalars (enums included) can be illustrated
with the following schema:

```GraphQL
union Number = Int | Float

type Query {
  number: Number
}

```

 and the query:

```GraphQL
query {
  number {                  # Problem #1: no selection sets for scalars
    ... on Int { ??? }      # Problem #2: how would we get the Int value
    ... on Float { ??? }    # ... or the Float one?
  }
}
```

There is also a challenge with programming languages used in server code - 
those that do not differentiate between `Int` and `Float`, for example, or 
`ID`, `String` and, perhaps other scalars.

Ultimately, such issues are solvable in server code and, if that does not appear
feasible to a particular server, nothing forces them to make use of unions that
could confuse their code. In other words, this is not GraphQL's problem to solve.

On the client, howers, there is a problem #3. Once the data arrives to the client,
other than  looking at the raw JSON value, they would not be able to detect the 
scalar's exact type, nor would they be able to request it, i.e.:

```GraphQL
query {
  number { __typename }
}
```

## Eliminated alternatives

Knowing that any and all clients and their requests only use object
(compound type) unions as that is all that existed so far, there is no 
fear of compatibility issues if we always use selection sets for
union-typed fields. This addresses the problem #1 as described in the
background.

Problem #2 can be addressed using the concepts described in 
[RFC: Safe navigation (shortcut) and refactoring operators](SafeNavigationAndRefactoring.md),
speficially the "trackback" `__back` field, though somewhat clumsy:

```GraphQL
query {
  number {
    ... on Int   { __back.number } # refers (back) to self from the container
    ... on Float { __back.number } # refers (back) to self from the container
  }
}
```
A special field / keyword could be introduced to make this less clumsy:

```GraphQL
query {
  number {
    ... on Int   { __value }
    ... on Float { __value }
  }
}
```

Note that it is possible to ignore (exclude) values for certain types:

```GraphQL
query {
  number {
    ... on Int   { __value }
    # Skip Float values, we only care about Ints
  }
}
```

Neither of this, however, addresses the problem #3. That problem is similar
to the one in [RFC: Input Unions](InputUnions.md) and shares the same root cause.
In that sense, to have consistent syntax we can leverage the same solution
in both cases. This requires a promotion of unions that have 
at least one scalar type choice to a special anonymous object type.

## Proposal

Using the original example:

```GraphQL
union Number = Int | Float

scalar IPv4Address

union Something = String | ID | IPv4Address

type Query {
  number: Number
  something: Something
}
```

... would be automatically treated as follows:

```GraphQL
union Number = Int | Float

type Query {
  number: type { 
    __typename: String!
    __value: union Int | Float # leaf, no special handling
  }
}
```

However, these types aren't entirely anonymous as described in 
[RFC: Anonymous types](AnonymousTypes.md) and require special 
as they actually inherit the handling of the base type, so the query can
continue to look the same as the last example, except for the inclusion of 
`__typename`:

```GraphQL
query {
  number {
    __typename
    ... on Int   { __value }
    ... on Float { __value }
  }
}
```


