# RFC: Comprehensive Enhancements [UNFINISHED DRAFT]

> ***This is a collection of coordinated changes that work with each other.***
> While it may be possible to split those into individual parts, the aim here
> is to present them together precisely to show how they work and enhance each
> other, making sure they don't conflict.

> **References, relates to or affects the following other RFCs:**
>
> - [RFC: GraphQL Composite Schemas](CompositeSchemas.md)
> - [RFC: GraphQL Input Union](InputUnion.md)
> - [RFC: Client Controlled Nullability](ClientControlledNullability.md)
> - [RFC: Schema Coordinates](SchemaCoordinates.md)
> - [RFC: Annotation Structs](AnnotationStructs.md) (previously [RFC: Metadata Structs](MetadataStructs.md))
> - [RFC: GraphQL Subscriptions](Subscriptions.md)
> - [RFC: GraphQL Defer and Stream Directives](DeferStream.md)


GraphQL has become sufficiently good to be in massive use while continuing to suffer from some major misses. May are subject to specific other proposals here which are great in their own right. The challenge with those is that each appears tightly focused to solving their own challenges. In that they appear to be:

- taking GraphQL in multiple different directions, and
- not taking advantage of solving multiple issues with single/smaller changes.

Examples of some of these misses include:

- Presence of only one global namespace halts schema composition, schema modules and standardization:
  - types (all kinds, input and output, scalar and object, including interfaces and unions)
  - fields (all kinds, including queries, mutations and subscriptions)
  - directives
- Execution order inconsistencies
- Often implied, partially specified restriction of mutations to only one (top) level
- Absence of introspectable schema (metadata) extensions as directives remain unexposed. Typical workardounds include alternate access to the schema that exposes directives.
- Lack of transaction markings, support
- Lack of immediate responses for subscriptions
- Unmet desire for in-data (embedded) errors often worked around using unions
- Inability to pass type references as values of input arguments in a way that can be validated by generic tools. Typical workarounds include use of enums but they can't (easily) be (re)generated upon schema composition.
- Complications around responding with anonymous instances of interface types
- Lack of input variants/unions (presently, see [RFC: GraphQL Input Union](InputUnion.md)).
- Terminology confusion: mutations are passed inside the `query` parameter and we're talking of `operationName` [as well](https://graphql.org/learn/serving-over-http/#post-request).

Intent of this RFC is to initiate and focus further GraphQL evolution. I'm taking a first stab at it here. Do note that this proposes multiple additions to the spec as the goal is to try to address multiple issues in a coherent way. I will set some goals and non-goals:

1. All existing and currently valid clients, servers and schemas must continue to be valid.
2. All existing concrete pairings of clients and servers must continue to function as well as they did even after the server is updated with newer GraphQL spec version support.
3. Clients updated to a newer GraphQL spec version should be able to discover the abilities of the server, ideally without having to request or know about the specific GraphQL spec or server version.
4. (Non-goal) Legacy servers do *not* need to be able to support requests that leverage GraphQL features from a newer GraphQL spec than they were built for.
5. (Non-goal) We can require legacy clients to change before they can leverage new GraphQL features as long as there is a workaround that servers cen employ to continue to support these clients.
6. (Non-goal) There are no provisions for IDEs and code generation tools - they must be updated 

# Iterative evolution

While this RFC is a monolith, it is so only to depict the related changes together. Though it would be nice to approve and apply them all at once, that is not required. It is OK to start with some and leave others for later. This also applies within a single complex change.

# Safe navigation (shortcut) and refactoring operators

Described in: [RFC: Safe navigation (shortcut) and refactoring operators](SafeNavigationAndRefactoring.md)



# Namespace support

Described in: [RFC: Namespace support](Namespacing.md)


# Allow directives on directives

Described in: [RFC: Allow directives on directives](DirectivesOnDirectives.md)


# Introspectable directives

Described in: [RFC: Introspectable directives](IntrospectableDirectives.md)


# Operation types

Described in: [RFC: Operation Types](OperationTypes.md)


# Transactions

Described in: [RFC: Transactions](Transactions.md)


# RFC: Regular expressions support

Described in: [RFC: Regular expressions support](RegularExpressions.md)


## Privacy and security directives

Described in: [RFC: Privacy and security directives](SecurityDirectives.md)


# Allow declarations of empty types

Described in: [RFC: Allow empty types](EmptyTypes.md)


# Permit interface types in concrete output

Described in: [RFC: Permit interface types in concrete output](InterfacesInOutput.md)


# Anonymous unions and in-data errors

Described in: [RFC: Anonymous unions and in-data errors](UnionsWithErrors.md)


# Input expressions that enable powerful DSLs

Described in: [RFC: Input expressions that enable powerful DSLs](InputExpressions.md)

