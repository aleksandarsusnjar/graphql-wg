# RFC: Allow declarations of empty types

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar)

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)


It is often useful to split the declarations of common types, such as
`Query`, `Mutation`, `Subscription` and `Operation` and others across multiple
files or schema sources. While this can be accomplished using `extend`, exactly one
of those must be chosen to serve as the seed that does not use the `extend` keyword.
This is not always possible, such as when there is nothing common to specify in the
seed and the souyrces are otherwise meant to be independent.

Allowing the ability to `extend` when there is no such seed is problematic as well 
as it may remove the detection of the expected declaration missing that existing
projects may rely on. Similarly, simply allowing empty declarations may miss accidental deletions.

To fix this the proposal is to introduce a new keyword, `empty`, to use as a prefix for
type declarations that have no fields. If that prefix is present, having the field block
should be considered an error.

Examples:
```GraphQL
empty type Query 
empty type Mutation
empty type Subscription
empty type graphql.Operation

empty interface SeedInterface

# Cannot directly declare an empty type implementing an interface
empty type Custom implements SomeInterface
# ... but that can be restated as follows:
empty type Custom
extend type Custom implements SomeInterface {
  ...
}

empty type Bad1 {}       # Error: field block present, though empty
empty type Bad2 { ... }  # Error: field block present
```