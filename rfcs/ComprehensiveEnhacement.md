# RFC: Comprehensive Enhancements [UNFINISHED DRAFT]

> ***This is a collection of coordinated changes that work with each other.***
> While it may be possible to split those into individual parts, the aim here
> is to present them together precisely to show how they work and enhance each
> other, making sure they don't conflict.

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
- Lack of input variants/unions
- Terminology confusion: mutations are passed inside the `query` parameter and we're talking of `operationName` (as well)[https://graphql.org/learn/serving-over-http/#post-request].

Intent of this RFC is to initiate and focus further GraphQL evolution. I'm taking a first stab at it here. Do note that this proposes multiple additions to the spec as the goal is to try to address multiple issues in a coherent way. I will set some goals and non-goals:

1. All existing and currently valid clients, servers and schemas must continue to be valid.
2. All existing concrete pairings of clients and servers must continue to function as well as they did even after the server is updated with newer GraphQL spec version support.
3. Clients updated to a newer GraphQL spec version should be able to discover the abilities of the server, ideally without having to request or know about the specific GraphQL spec or server version.
4. (Non-goal) Legacy servers do *not* need to be able to support requests that leverage GraphQL features from a newer GraphQL spec than they were built for.
5. (Non-goal) We can require legacy clients to change before they can leverage new GraphQL features as long as there is a workaround that servers cen employ to continue to support these clients.
6. (Non-goal) There are no provisions for IDEs and code generation tools - they must be updated 

# Iterative evolution

While this RFC is a monolith, it is so only to depict the related changes together. Though it would be nice to approve and apply them all at once, that is not required. It is OK to start with some and leave others for later. This also applies within a single complex change.

# Safe navigation (shortcut) operator `?.`

This is a convenience feature that can help in cases when complex types with many properties are queried by clients who only need some.
It is also leveraged to addresses the field namespacing challenges noted later.

The goal is to allow usages of this operator only in field selections with an explicit alias, such as:

```
  <alias> `:` <field-ref-1> `?.` <field-ref-2> `?.` <field-ref-3> <opt-selection>
```

This would yield the result `<alias>` to be:

- `null` if any `<field-ref-#>` in the path yields `null`,
- otherwise, if all `<field-ref-#>` prior to the last are ***not*** lists, the output of `<field-ref-3> <opt-selection>` only, without nesting into `<field-ref-1>` and `<field-ref-2>`.
- otherwise, a List of all individual outputs of `<field-ref-3> <opt-selection>`, without nesting into `<field-ref-1>` and `<field-ref-2>`.

Example:
```GraphQL
query {
  example: foo?.bar?.far { number }
}
```
... could yield:

```json
{
  "data": {
    "example": { "number": 1 }
  }
}
```

or, if either `foo` or `bar` returns a list:

```json
{
  "data": {
    "example": [{ "number": 1 }, { "number": 2 }, { "number": 3 }]
  }
}
```

# Namespace support

This is a complex topic discussed before. Proposal here differs a bit to take into account additional and different experience of the author(s).

## Identifiers, `.` (dot) as a namespace delimiter

To remain compatible with every possible valid implementation of prior specs, we cannot unambiguosly reuse a character that is already permitted in GraphQL identifiers within those identifiers. We could opt to split the namespace from the name in more profound ways that would not rely on this but it would yield syntax that is more cumbersome to use. Software developers using many different programming languages are already well used separating their namespaces or package names using `.` so this plays well with familiarity, readability and the learning curve.

This change introduces the `.` character as a valid part of GraphQL namespaced identifiers but with explicitly stated semantics - namespaces.

Currently, the (October 2021 GraphQL spec)[https://spec.graphql.org/October2021/#sec-Names] states:

```
Name ::
  - NameStart NameContinue* [lookahead != NameContinue]

NameStart ::
  - Letter
  - `_`

NameContinue ::
  - Letter
  - Digit
  - `_`

Letter :: one of
  - `A` `B` `C` `D` `E` `F` `G` `H` `I` `J` `K` `L` `M`
  - `N` `O` `P` `Q` `R` `S` `T` `U` `V` `W` `X` `Y` `Z`
  - `a` `b` `c` `d` `e` `f` `g` `h` `i` `j` `k` `l` `m`
  - `n` `o` `p` `q` `r` `s` `t` `u` `v` `w` `x` `y` `z`

Digit :: one of
  - `0` `1` `2` `3` `4` `5` `6` `7` `8` `9`
```

This would remain unchanged but be augmented by:

```
NamespaceName ::
  - NamespaceStart NamespaceContinue* [lookahead != NamespaceContinue]

NamespaceStart ::
  - Letter

NamespaceContinue ::
  - Letter
  - Digit

FullyQualifiedNamespace ::
  - (NamespaceName `.`)* NamespaceName
  - `__`

FullyQualifiedIdentifier ::
  - (NamespaceName `.`)+ Name

UnqualifiedIdentifier ::
  - Name
  
Identifier ::
  - FullyQualifiedIdentifier
  - UnqualifiedIdentifier
```

From this point on all relevant references to `Name` (scalar, enum, union, interface, object type, input type and directive names) should be replaced with `Identifier`. This also applies to field names but there are additional considerations noted later.

A single-character `__` is specially treated as a reference to the root namespace.

Two identifiers are considered equal only if they are entirely equal taking into account the namespace. An unqualified identifier is subject to the default namespace within its scope, noted separately.

Examples:

```GraphQL
scalar inet.IP4AddressLiteral
enum org.iso.iso3166_1.ThreeLetterCountryCode {
    ...
}
object org.example.Computer {
  ip4Address: inet.IP4AddressLiteral
  countryCode: org.iso.iso3166_1.ThreeLetterCountryCode
}
```

## Default namespaces

This is an optional but convenient feature that can increase readability of GraphQL schemas by making it is less verbose. 

### `namespace` block 

A new `namespace <FullyQualifiedNamespace> { ... }` declaration is introduced to eliminate common namespace repetition.

```GraphQL
namespace org.example {
  namespace nested {
    # Effective namespace here is org.example.nested
  }
}
```

To avoid or lessen the risk of hard-to-handle conflicts within a single schema, the fully qualified namespace identifiers must not equal any other scalar, enum, interface, output type, union, input type or directive fully qualified identifier. Additionally, it is advised that:

- Namespace identifiers are all lowercase (more precisely do ***not include uppercase letters***). This can be enforced.
- Unqualified (simple) names of scalars, enums, interfaces, output types, unions and input types begin with an uppercase letter. This cannot be enforced without potentially breaking compatibility but is in line with out-of-box type names.
- Unqualified (simple) names of directives begin with a lowercase letter, to reduce variation while keeping previously defined standard directives valid as per the convention.

Above conventions can also aid in achieving additional potentially beneficial distinction - that no fully qualified namespace identifier matches any effective fully qualified field identifier (described later). 

### Default type and directive declaration namespace

If `scalar`, `enum`, `union`, `interface`, `object`, `input`, `directive` and (new) `namespace` declarations use unqualified identifiers then the assumed namespace is the one declared in the containing (new) namespace block.

Example:

```GraphQL
namespace org.example {
  namespace nested {
    object Computer { # Effective identifier is org.example.nested.Computer
      ...
    }
  }
}
```

### Default type and directive reference namespace

If `scalar`, `enum`, `union`, `interface`, `object`, `input` and directive declarations use unqualified identifiers then the assumed namespace is the one declared in the nearest containing (new) namespace block that contains the referenced declaration.

Example:

```GraphQL
scalar Speed 
namespace org.example {
  scalar Speed  # Effectivelly org.example.Speed, hides "root" Speed
  namespace nested {
    object Computer { # Effective identifier is org.example.nested.Computer
        speed1: __.Speed           # Effective type is `Speed`, bypasses defaulting
        speed2: org.example.Speed  # Effective type is `org.example.Speed`, explicit
        speed3: Speed              # Effective type is `org.example.Speed`, defaulted
                                   # (there is no `org.example.nested.Speed`)
    }
  }
}
```

The following introspection schema changes are needed:

```GraphQL
extend type __Schema {
  namespaces: [graphql.Namespace!]!
}

type graphql.Namespace {  
  identifier: ID!
  superspace: Namespace    # optional
  subspaces: [Namespace!]! # optional
  types: [__Type!]!
  directives: [__Directive!]!
}

extend type __Type {
  """
  Fully qualified identifier
  """
  id: ID!

  """
  Namespace this type is in.
  """  
  namespace: graphql.Namespace!
}

extend type __Directive {
  """
  Fully qualified identifier
  """
  id: ID!

  """
  Namespace this directive is in.
  """  
  namespace: graphql.Namespace!
}
```

### Default field declaration namespace

When composing large schemas of not entirelly coordinated projects, such as when "stitching" or "federating" them parts of schema obtained from different sources could introduce what would so far appear as the same field with conflicting definitions. Some ways to address this are to either rename those fields at the source (often impossible) or to apply some sort of runtime renaming (that introduces incompatibilities and may not be easy). 

Additionally, there exist property naming standards, such as [Dublin Core](https://www.dublincore.org/specifications/dublin-core/dcmi-terms/) and it would be useful to be able to encode these in a standard, modular fashion in GraphQL.

Using fully qualified names for fields would be cumbersome for a variety of reasons:

- Problems with default aliases due to additional characters in the name or ambiguity if namespace identifiers are omitted
- Verbosity reducing readability

There is also a question of whether the namespace of the field is a distinct concept from the containing/declaring type. Presently the namespace of the field is effectivelly the (name of the) outermost declaring type - e.g. the `interface` that declared it first. However, we can already spot the problem. Multiple interfaces *can* declare the same-name field and it is to be treated as a single field as long as nothing else causes this to be impossible - otherwise it is an error. This is behaviour must be preserved for compatibility while it is also a problem that must be solved in the context of large composite schemas where those must be treated as distinct fields not to be "fused" into one.

Example:
```GraphQL
interface people.Person {
  name: String!
  ...
}
interface commercial.Business {
  name: String!
  ...
}
object example.SoleProprietor implements Person, Business {
    # What to do with the name?
    # Personal and business names need not be equal.
}
```

Please note that this is just an illustration. Conflicts such as these can and are avoided presently by (re)factoring the schema differently. The point here is that this is not always easily possible as these types could be coming from distinct projects that aren't coordinated and conflicts can be introduced when least expected.

The proposal here is to equate the field namespace to the declaring type while allowing explicit specification whether the field declaration is meant to be additional or a redeclaration. This can also help in early detection of issues with incompatible upstream changes.

Consider the following example:

```GraphQL
# From project "a"

interface a.A {
  id: ID!
  a: String
}

# From project "b"

interface b.B {
  id: ID!
  b: String
}

# Project "c"

object c.C implements a.A, b.B {
  id: ID!   # Comes from both a.A and b.B
  a: String # First declared in a.A
  b: String # First declared in b.B
  c: Int
}
```

... then project "b" updates its definition to:

```GraphQL
interface b.B {
  id: ID!
  b: String
  c: String
}
```

... bringing in a property `c` that is incompatible with (`String` vs `Int`) but distinct from that one in `c.C`.

Our goal is to find a way to make the initial state work without changes and the new state workable with only changes to project c and no changes to the clients.

Note the following:

1. We will assume that a client will only be exposed to newer schemas only after all upstream projects are given the change to do this first.
2. Due to (1), all existing users/clients of the field `c` only knew about it from `c.C` and not from `b.B`.
3. The clients may be updated to also know about `c` from `b.B` but, by that point, they would also have the new `c` schema.
4. Due to `a.A` and `b.B` independently defining `id`, we cannot avoid it having multiple namespaces, though it would be nice to have one. Given coordination, those two interfaces could both be based on another, common interface, defining common fields.
5. Project `c` needs to be clarify and distinguish inherited from those that are not and are thought as distinct. This concept is similar to specifying whether a method declaration is an override or not in OOP languages.

Here we will modify the following:

```
FieldDefinition : Description? Name ArgumentsDefinition? : Type Directives[Const]?
```

to:
```
FieldDefinition : Description? FieldSource? FieldIdentifier ArgumentsDefinition? : Type Directives[Const]?

FieldSource : 
  - `inherited`
  - `uninherited`

FieldIdentifier : Identifier
```

> Note that `Identifier` is either `FullyQualifiedIdentifier` or `UnqualifiedIdentifier`.


The reason for `FieldSource` keywords is to avoid defining new directives that may conflict with existing, though that approach could work as well. Additionally, those influence schema processing in ways that out-of-box directives did not so far and need to be exposed to downstream schema processing tools to be able to do the same logic as the initial parser, as follows:

1. If the defined using an unqualified identifier:
  a) if it is marked `uninherited` the effective namespace set will only contain the declaring type.
  b) if it is marked `inherited` then we need to identify which fields are being inherited, by matching only the unqualified/last part of the identifier. This may result in multiple fields that are already marked as distinct by the implemented interfaces (i.e. one interface implementing/extending another, declaring an uninherited field). This situation can only occur when this enhancement is used and can, thus, result in an error - prohibition of merging those multiple fields into one. If there are no such obstacles and merging is valid as per the existing spec, then the resulting set of effective namespace set will contain the union of all effective namespaces of all inherited fields plus the declaring type.
2. If the defined using a fully qualified identifier
  a) if it is marked `uninherited` but there is at least one inherited same-name field with matching namespace identifier, error out. If there are none, the effective namespace of the field is as explicitly stated plus the containing type, if distinct.
  b) if it is marked `inherited` then we need perform the same search as in (1b) while (initially) ignoring inherited fields that do not have the explicitly stated namespace. We then expand this set to include other inherited same-name fields with overlapping namespaces to those already matched and repeat the process until no more are found. If a conflictg as in (1b) is discovered, an error is to be reported.
3. By this time, as an extra validation step, we can confirm that there are no remaining distinct fields that have some equal fully qualified names (note that each field can have multiple) and we also check that each inherited field is matched by exactly one declaration.

Introspection schema changes required:

```GraphQL
enum graphql.FieldInheritance {
    AUTO_INHERITED
    INHERITED
    AUTO_UNINHERITED
    UNINHERITED
}

extend type __Field {
  name: String!
  namespaces: [graphql.Namespace!]!
  ids: [ID!]! # namespaces.id + '.' + name
  inheritance: graphql.FieldInheritance! 
  description: String
  args: [__InputValue!]!
  type: __Type!
  isDeprecated: Boolean!
  deprecationReason: String
}
```

With this, the initial example can be annotated as follows:

```GraphQL
# From project "a"

interface a.A {
  id: ID!    # Effectivelly a.A.id
  a: String  # Effectivelly a.A.a
}

# From project "b"

interface b.B {
  id: ID!   # Effectivelly b.B.id
  b: String # Effectivelly b.B.b
}

# Project "c"

object c.C implements a.A, b.B {
  id: ID!   # Effectivelly a.A.id, b.B.id and c.C.id
  a: String # Effectivelly a.A.a and c.C.a
  b: String # Effectivelly b.B.b and c.C.b
  c: Int    # Effectivelly c.C.c
}
```

After project "b" updates its definition to:

```GraphQL
interface b.B {
  id: ID!   # Effectivelly b.B.id
  b: String # Effectivelly b.B.b
  c: String # Effectivelly b.B.c
}
```

... project "c" can update its definition to:

```GraphQL
object c.C implements a.A, b.B {
  inherited   id: ID!    # Effectivelly a.A.id, b.B.id and c.C.id
  inherited   a: String  # Effectivelly a.A.a and c.C.a
  inherited   b: String  # Effectivelly b.B.b and c.C.b
  inherited   c: String  # Effectivelly b.B.c
  uninherited c: Int     # Effectivelly c.C.c
}
```

... or, as an alternate example:

```GraphQL
object c.C implements a.A, b.B {
  inherited   id: ID!        # Effectivelly a.A.id, b.B.id and c.C.id
  inherited   a.A.a: String  # Effectivelly a.A.a and c.C.a
  inherited   b.B.b: String  # Effectivelly b.B.b and c.C.b
  inherited   b.B.c: String  # Effectivelly b.B.c
  uninherited c: Int         # Effectivelly c.C.c
}
```

## Default output field reference namespace

In clients's requests the fields are referenced as follows:

```
Field : Alias? Name Arguments? Directives? SelectionSet?
```

This should be changed to:
```
Field : Alias? FieldIdentifier Arguments? Directives? SelectionSet?
```

> Note that `FieldIdentifier` is an `Identifier`, which is either `FullyQualifiedIdentifier` or `UnqualifiedIdentifier`.

If the field reference identifier is fully qualified, then the matching field is the one that has a matching pair of one of its namespaces and name that would result in the same fully qualified identifier. 

If the field reference identifier is unqualified then the assumed namespace is the contextual output type (interface or object) whether implied or explicitly stated in fragments. The matching then proceeds the same way as with the fully qualified identifier.

Example based on the declaration in the previous section:

```GraphQL
query {
  getC {  # outputs c.C
    id    # Effectivelly c.C.id, thus matching the merged id
    a     # Effectivelly c.C.a, matches inherited a
    ... on b.B {
      b   # Effectivelly b.B.b, matches inherited b
    }
    c     # Effectivelly c.C.c, matches UNINHERITED c
    b.B.c # Effectivelly b.B.c, matches c INHERITED from b.B
  }
}
```

If the field declared inside the type uses an unqualified identifier then the assumed (default) namespace for it is equal to the namespace of the declaring type.

In cases when the field with the same simple name is (also) declared within an interface implemeted by the declaring type the schema designer can:

  1. Continue to leverage the default if both the interface in question and the declaring type are in the same namespace.
  2. Use fully qualified identifiers to disambiguate.

In the following example:

```GraphQL
object org.example.Computer {
  ip4Address: inet.IP4AddressLiteral
  countryCode: org.iso.iso3166_1.ThreeLetterCountryCode
}
```

Both `ip4Address` and `countryCode` should be considered as within the `org.example` namespace and, thus:

## Input field declaration namespace

Input field declarations can follow the same rules as the output, while presently not having to deal with inheritance.
Should input type inheritance be introduced the same logic can apply as with the output types.

## Input field reference namespaces

The same rules can apply as for the output fields but without having to deal with fragments or inheritance, until it is introduced.

## Namespace imports

Namespace "import" feature of some programming languages can be convenient here as well. However:

- It could be thought of as a preprocessing step to avoid further complexity in namespace management.
- Local convenience conventions should not affect the exposed result.
- It must not be able to fully "hide" (conflict) with any existing global or local (imported) declaration or reference
- Its scope must be entirely clear, ideally even if the schema is split into multiple unordered fragments

Proposed approach:

```
SchemaImport :: `with` Import (`, ` Import)* `{` SchemaImportBody `}`

RequestImport :: `with` Import (`, ` Import)* `{` RequestImportBody `}`

Import : FullyQualifiedIdentifier (`as` FullyQualifiedIdentifier)
```

Example schema:
```GraphQL
with inet.IP4AddressLiteral, org.iso.iso3166_1 as iso {
  namespace org.example {
    object Computer {
      ip4Address: IP4AddressLiteral
      countryCode: iso.ThreeLetterCountryCode
    }
}
```

Example request:
```GraphQL
with org.example {
  query {
    getDevices {
      ... on Computer {
        ip4Address
        countryCode
      } 
    }
  }
}
```

## Reserved namespaces

- `__` - explicit root namespace
- `graphql` - reserved space for future needs while avoiding potential conflicts

# Allow directives on directives

Adding directives to directives will allow various schema meta-processing tools to introduce their own additional special processing even for directives they do not define.

Change:

```
DirectiveDefinition : Description? directive @ Name ArgumentsDefinition? `repeatable`? on DirectiveLocations
...
TypeSystemDirectiveLocation : one of
  - `SCHEMA`
  - `SCALAR`
  - `OBJECT`
  - `FIELD_DEFINITION`
  - `ARGUMENT_DEFINITION`
  - `INTERFACE`
  - `UNION`
  - `ENUM`
  - `ENUM_VALUE`
  - `INPUT_OBJECT`
  - `INPUT_FIELD_DEFINITION`
```

to (append `Directives` to DirectiveDefinition and append `DIRECTIVE_DEFINITION` to locations):

```
DirectiveDefinition : Description? directive @ Name ArgumentsDefinition? `repeatable`? on DirectiveLocations Directives
...
TypeSystemDirectiveLocation : one of
  - `SCHEMA`
  - `SCALAR`
  - `OBJECT`
  - `FIELD_DEFINITION`
  - `ARGUMENT_DEFINITION`
  - `INTERFACE`
  - `UNION`
  - `ENUM`
  - `ENUM_VALUE`
  - `INPUT_OBJECT`
  - `INPUT_FIELD_DEFINITION`
  - `DIRECTIVE_DEFINITION`
```

and append

# Introspectable directives

There have been multiple attempts to expose additional/custom schema metadata inside introspection in a variety of ways. Meanwhile, many ended up communicating the schema directly, bypassing introspection, to gain access to directives. The proposed approach can make the status quo valid in a clean way, by defining a new directive:

```
"""
Exposes the annotated element (directive) in introspection.
"""
directive @graphql.introspectable on DIRECTIVE_DEFINITION
``` 

Additional introspection schema is required:

```GraphQL
extend type __Schema {
   schemaDirectiveDefinitions: [graphql.SchemaDirectiveDefinition!]!
}

extend type __Type {
  schemaDirectives: [graphql.SchemaDirective!]!
}

extend type __Field {
  schemaDirectives: [graphql.SchemaDirective!]!
}

extend type __EnumValue {
  schemaDirectives: [graphql.SchemaDirective!]!
}

extend type __Directive {
  schemaDirectives: [graphql.SchemaDirective!]!
}

namespace graphql {
  type SchemaDirectiveDefinition {
    name: String!
    id: ID!
    namespace: Namespace
    description: String
    locations: [__DirectiveLocation!]!
    args: [__InputValue!]!
    isRepeatable: Boolean!
  }

  type SchemaDirective {    
    definition: SchemaDirectiveDefinition
    description: String
    locations: [__DirectiveLocation!]!
    args: [__InputValue!]!
    isRepeatable: Boolean!
  }
}
```

# Operation types

Present operation division into queries, mutations and subscriptions is noble in intent but is causing multiple challenges:

- Clarity about what constitutes side-effects
- Basic features with side-effects (e.g. authentication with session creation) may be required to perform simple read-only (side-effect-free) fetches. This pushes even those fetches into the mutation space. Alternatively, one could ignore their side-effects and expose them as queries, too. That would result in duplication of similar or equal functionality into both query and mutation spaces.
- There are arbitrary inconsistencies that do not seem to match the clarity of the rest of the GraphQL spec, namely that only mutations are subject to execution ordering specificiations and are prevented from nesting. 
- Prevention of mutation nesting forces all possible combinations of mutations to have to be exposed individually. While namespacing may help make this navigable, it nevertheless produces much larger schemas than would be needed if compositing mutations were possible. For example, consider the basic authentication or transaction boundary mutation having multiple piecewise mutations within.
- Real-world notificatation subscription negotiation process often involves the need to respond with some data immediatelly back, but there is no way to specify that for subscription requests.
- Subscriptions cannot be nested either. Of course, it does not make sense to nest one subscription inside another subscription, but nesting them within authentication mutations or transaction or other contextual specifications does.
- Mutations often have nothing to yield/output beyond success or error indication but are presently forced to. Returning a scalar is not recommended as it cannot be compatibly extended later and cannot leverage the common approach for embedded errors either, forcing immediate requirement for dedicated complex output types. Since those types cannot be empty either, often dummy fields are included and clients are forced to select at least one even if they don't care about them.

The aim is to maintain the following:

- Ability to indicate and discover that operation does not have side-effects (present queries).
- Ability to indicate and discover that operation does have side-effects (present mutations).
- Ability to specify the immediate outputs available and select from these (present queries and mutations).
- Ability to have an understandable execution order specification.
- Ability to specify the deferred/notification outputs available and select from these (present subscriptions).
- Compatibility with existing schemas, servers and clients.

The aim is NOT to attempt to introduce automatic compatibility mapping between the new features introduced here and the legacy. A client wishing to access the new features of a server leveraging the enhancements documented here would have to also accept these changes.

## New operation type

To leave the semantic challenges outlined earlier, present queries, mutations and subscriptions become legacy concepts that we avoid making changes to. Instead we focus on a single new operation type, `graphql.Operation` that should address all these challenges within. We want:

- The ability to indicate whether there are side-effects.
- The ability to control order and cardinality of execution.
- The ability to control immediate outputs, including returning "nothing yet".
- The ability to control deferred outputs.

For this we introduce the following:

- The new `graphql.Operation` type, analoguous to present `Query`, `Mutation` and `Subscription`, which must be maintained but should be considered deprecated.
- The new `operation` keyword for requests, analoguous to present `query`, `mutation` and `subscription`, accepting names and input variables just as they do. The use of `query`, `mutation` and `subscription` must be maintained but should be considered deprecated. The `operation` keyword acts as a root access point to the `graphql.Operation` singleton. 
- Similarly to how presently the keyword `query` can be omitted from requests if they entirely consist of just a single query, the same can be applied to operation with two caveats:
  - Requests beginning with `{` have to continue to default to `query` (queries) for compatibility, but only if the `Query` type exists. In this proposal it does not need to exist.
  - Requests beginning with `[` (described later) default to `operation`.
- Present/prior nesting and execution order specifications for queries, mutations and subscriptions do not apply at all and are replaced with new ones here.
- New syntax and additional considerations are introduced for immediate and deferred output control.

As per this proposal, an operation is nothing but a GraphQL (output type) field. It is proposed that the term "field" is also replaced with the term "function" going forward. 

## No output

Some operations do not have natural outputs beyond success/error status and should be able to rely on the GraphQL error system to communicate that. It is presently not possible to mark fields in a way to indicate they have no outputs and there is no way for the clients to cleanly request such fields in a forward compatible manner.

This change introduces a `graphql.NothingYet` type which would be similar to `Void`, `void` or `None` types in some programming languages but with a clear indication that this must be treated as a temporary situation that may change.

The clients should be able to invoke any field without requesting any of any output that may be provided. This should be marked clearly. Proposal here is to use `?` in place of the field selection block or equivalent spot for scalar fields. This indicates that the client is interested in executing the function represented by the field but only wants to be informed of errors, if any. 

Example... Given:

```GraphQL
type graphql.Operation {
  pour(content: String): graphql.NothingYet
}
```
A valid request example can be:

```GraphQL
{
  pour(content: "Stuff")?
}
```
... as `graphql.NothingYet` has no value, scalar or complex/object. That request continues to work even if the server/schema is updated to:

```GraphQL
type graphql.Operation {
  pour(content: String): PourOutput!
}
```

## Ordering

There are multiple different kinds of ordering considerations immediatelly apparent in GraphQL:

1. ***Execution ordering***. Presently any (e.g. queries) or serial (mutations).
2. ***Collection value ordering***. Presently GraphQL only supports ordered lists, not unordered sets, but existing implementations use this for unordered collections as well.
3. ***Output field ordering***. Presently always intended to match the request field order, though fragments can affect this in potentially non-trivial ways.

The approach recommended here is to make multipurpose improvements to GraphQL that can be leveraged for the new operation type as well.

### Collection value ordering

Current syntax to denote list types is to wrap the element type in square brackets. Specification text:

> A GraphQL list is a special collection type which declares the type of each
> item in the List (referred to as the *item type* of the list). List values are
> serialized as ordered lists, where each item in the list is serialized as per
> the item type.
>
> To denote that a field uses a List type the item type is wrapped in square brackets
> like this: `pets: [Pet]`. Nesting lists is allowed: `matrix: [[Int]]`.

To this we can add an unordered "Bag" collection type that still allows element duplication.
This is denoted similarly to lists, but using curly braces to wrap the element type, i.e. `pets: {Pet!}`.
Nesting is similarly permitted: `groups: {{Person!}}`. Using `{` is possible because in this position 
it cannot be confused with anything else. Choice in influenced by JSON, JavaScript and, to a lesser extent, 
even GraphQL object literal notation differences from array or list literals.

This approach can be later enhanced to include the uniquiness criteria, thus identifying sets.
Examples to iillustrate this without going into a full proposal:

- `groups: {{Person!:(id),(email)}}`
- `groups: {{Person!}} @graphql.distinctBy(keys: ['id']) @graphql.distinctBy(keys: ['email'])`
- `pets: {Pet!:(id),(species,breed,name),(owner?.id,name)}`

### Execution ordering

GraphQL is predominantly leaving the execution order unstated and up to the server to choose.
The only exception to this are the (root, top-level) mutations. See [6.2.2 Mutation](https://spec.graphql.org/October2021/#sec-Mutation):

> If the operation is a mutation, the result of the operation is the result of executing the operation’s top level selection set on the mutation root object type. This selection set should be executed serially.

We will assume the following:

1. All servers can implement ordered execution.
2. Parallel/concurrent execution may be possible and implemented but never required, even if requested.
3. Clients don't and should not care about how the servers accomplish their tasks as long as the outcome is as requested.
4. Clients should not ever require parallel execution as both server implementations and runtime conditions may get in the way.
5. Clients may, however, require and need to request serial execution for a sequence of operations that depend on side-effects of previous operations. Servers have to be able to honor such requests. This is beyond an ignorable hint and cannot rely on ignorable elements such as client directives.
6. We do not want arbitrary yet rigid context-sensitive specification such as presently specified mutations.

Presently all operations are nested within curly braces `{`...`}` and, as noted, allow parallel execution everywhere except for the top level mutations. We will leverage that as the default and will introduce different markings for cases when serial execution is requested. For that we will use the JSON/JavaScript/GraphQL/etc-inspired ordered collection notation using the square brackets `[`...`]` instead of the curly braces. This syntax has to be supported within `graphql.Operation` operations to any nesting depth. Support for legacy operation types/contexts (query, mutation, subscription) is possible but not required as it does not really help there.

Example:

```GraphQL
[
  doSomething(...) 
  doMore(...)
  finish(...)
]
```

is a shorthand for / equivalent to:

```GraphQL
operation [
  doSomething(...) 
  doMore(...)
  finish(...)
]
```

and requests that the execution of `doSomething`, `doMore` and `finish` is done in stated order and not in any other or in parallel. This is in contrast to the following which lets the server use/apply any ordering.

```GraphQL
operation {
  doSomething(...) 
  doMore(...)
  finish(...)
}
```

> Additional note: servers that can determine inter-dependency of individual steps (or lack thereof) can
> adjust the order and use parallel execution even when `[`...`]` is used, similar to how 
> [superscalar processors](https://en.wikipedia.org/wiki/Superscalar_processor) do the same.
> They could also examine `{`...`}` blocks the same way but this is not required and not to be expected.
> It is the clients' responsibility to understand implications of combining operations and use `[`...`]`
> if unsure. IDEs and validation tools can issue warnings if operations with side-effects are nested 
> within the same `{`...`}` blocks but should not fail as this is actually OK in most cases.
> Those warnings may be suppressed using client directives.

### Complex ordering and multiple blocks using anonymous fragments

The same block order control syntax also applies to a new concept of anonymous fragments that follow `...` directly with either a `[` or `{` block. The following example two arbitrary-execution-orded blocks that have to be executed one after another:

```GraphQL
operation {
  complex(...) [      
    ... { a1 a2 a3 }
    ... { b1 b2 b3 }
  ]
}
```
This ensures that `a1`, `a2` and `a3` will be executed before any of the `b1`, `b2` or `b3` but allows any order within each block. For example, the order may be `a3`, `a1`, `a2`, `b2`, `b1`, `b3`. Also all the `a#` can be executed in parralel, followed by all `b#` once `a#` are done. Note that this does not affect the desired output field order which should match the request - that is a separate issue.

### Controlling output field ordering

To indicate that any output field order is acceptable (separate from the execution order), we must have the ability to mark either the entire block as not requiring field order or have the means to indicate that order stops being important after a few "header" fields. For example, it may be important to get the `__typename` and `id` first, followed by anything and everything else. As directives are attached to elements and not the space between elements and fragment nesting and field merging can make directives additionally unsuitable.

A syntactic element, a symbol or a keyword is needed to mark this. Here `*` is proposed as it is easy to type on most keyboards and is often used as a wildcard implying flexibility. Multiple proposed alternatives follow:

#### `*` as a block separator

The asterisk `*` is chosen to mean "output field order is not important after this point" and is placed between the fields and/or fragments in the requests. For example:

```GraphQL
operation {
  subject(id: 123) {      
    __typename
    id
    *
    name
    location
  }
}
```

... would mean that `__typename` and `id` need to be output in that specific order but can be followed by any order of `name` and `location`.

This alternative can be easily modified if so decided to use a different, perhaps multi-character separator or keyword, such as `***`. Do note that keywords cannot be used here as they may end up conflicting with field names and directives would be attached to a field, not the space between, and may move together with the field during merging.

#### `*` for the entire block

Consider the simplification of the previous alternative where `*` is only permitted to be at the very beginning of a block and nowhere else. The same outcome can be accomplished using anonymous (or other) fragments:

```GraphQL
operation {
  subject(id: 123) {
    __typename
    id
    ... {*
      name
      location
    }
  }
}
```

## Side-effect marking (new mutations)

Any operation or function (a.k.a field) can be marked as having side-effects or not. With operations this is mostly a documentation exercise and can be annotated using a dedicated directive, such as:

```GraphQL
directive @graphql.affects(
  scopes: {ID!}!
) on FIELD_DEFINITION
```

A related directive can be used to detect operation inderdependencies:

```GraphQL
directive @graphql.uses(
  scopes: {ID!}!
) on FIELD_DEFINITION
```

... such that having at least one common scope in one operations "affects" and another operations "uses" list means that the second operation could be affected by the first one. The scope ids are arbitrary and up to the server / schema designer. Proxy servers / schema aggregation tools may transform those ids, e.g. by prefixing them, so the servers should not rely on clients getting the same ids.

Example:

```GraphQL
type graphql.Operation {
  getPerson(id: ID!): Person @graphql.uses(scopes: ["db:people"])
  newPerson(input: PersonInput!): Person @graphql.uses(scopes: ["db:people"]) @graphql.affects(scopes: ["db:people"])
  getGroup(id: ID!): Group @graphql.uses(scopes: ["db:groups"])
}
```

Ignoring nested invocations with their effects, we can see that `newPerson` could affect `getPerson` but not `getGroup`.


## Redirected outputs, streams / channels (new subscriptions)

With this proposal the way to replace legacy subscriptions is to use fields that have no immediatelly returnable values but will send or keep sending those values, as they become available, to a contextually specified (documented) destination which we may call a stream, channel, pipe or pipeline. Actual destinations are not the topic of this proposal and are intentionally left for the API designed to specify. It would, however, be nice to have a standard how to do this with GraphQL/HTTP specifically.

Here we want to differentiate betwen the two possibilities:

- Single-shot output. The channel can be automatically terminated after the single payload is sent.
- Multi-shot output. Many payloads can be sent through the channel.

As this changes the kind of the field, we can indicate this in the field type specification. Consider the following three examples:

```GraphQL
  regular:       [String!]!
  singleShot  -> [String!]! # or, alternatively, singleShot:  -> [String!]!
  packStream  => [String!]! # or, alternatively, packStream:  => [String!]
  valueStream =>  String!   # or, alternatively, valueStream: =>  String!
```

This approach opens up the possibility to handle more complex subscription scenarios. Given the following:

```GraphQL
type graphql.Operation {
  createTopicAndSendUpdates(entityId: 123): UpdateConnection
}

type UpdateConnection {
  subscription: SubscriptionDetails
  stream => UpdateMessage!
}

type SubscriptionDetails {
  id: ID!
  topic: URL!
}

type UpdateMessage {
  ...
}
```

... a client could do something like this

```GraphQL
{
  createTopicAndSendUpdates(entityId: 123) {
    subscription {
      id     # id to use for subscription management
      topic  # details of the auto-created topic
    }
    stream => { # => is here to clearly indicate what is going on, avoid any risk of confusion
        ...
    }
  }
}
```

At any point of time later the client could invoke the following to directly unsubscribe:

```GraphQL
{
  unsubscribe(subscriptionId: 123)?
}
```

The same approach could be used to do any other applicable subscription management, configuration, etc.

Also note that multiple streams/channels are possible within the same context, leaving the details to the designer.

It is recommended that properly namespaced types are created by the stakeholders / providers of each transport technolgy such as HTTP async delivery, WebSockets, Apache Kafka, GCP PubSub, etc. as these can then be detected and leveraged by a variety of the tooling including aggregating proxies.

# Transactions

Given the remainder of this proposal, single-phase transactions can be implemented as custom transaction operations having individual fields representing supported actions. This accounts for the possibility to have different transaction kinds which may not be combinable by the system. Example:

```GraphQL
type graphql.Operation {
  transaction: Transaction
}

type Transaction {
  action1(...): ...
  action2(...): ...
  action3(...): ...
  ...
}
```

The clients invoke this as any other operation, enforcing order as needed, e.g.:

```GraphQL
[
  transaction [
    action3(...) ...
    action1(...) ...
    action2(...) ...
  ]
]
```

This, however, does not help with distributed transactions. Those we can address better with [two-phase commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol) support. To do so we need:

1. An approach for the server to identify which operations can participate in two-phase commits.
2. An approach to separete the preparation from the completion phase into separate requests.

Note that the entire transaction must be described in the preparation phase but none of the resulting output can be expected until completion. This is distinct from both immediate outputs of standard operations, queries and mutations and also from subscriptions and streams, so a new syntax is needed to denote this. Having a clear standard is preferred for those tools that distribute transactions in the first place. We'll use the `prepare` and `commit` keywords for that, to be used at the top level of the request.

```GraphQL
prepare ExampleTransaction(args...) {
  # standard body as for a normal operation
}
```

The response to the above will only include a transaction id, of type `ID` and nothing from the field selection set. That id must be stored in a dedicated response area identified by the preparation name, e.g.: 

```json
{
  "transactions": {
    "ExampleTransaction": "fdeda22b-7d56-4138-9fe8-c35a575cd466"
  }
}
```

Once all the distributed transaction participants have positively "voted", the caller can then invoke the final commit:

```GraphQL
commit(transactionId: "fdeda22b-7d56-4138-9fe8-c35a575cd466")
```

Despite the lack of the field selection set, the expectation is that the response will include all the data as specified during preparation. As it is possible to have conflicts between multiple transaction outputs, multiple alternatives are offered:

## Separate output area keyed by transaction ids

```GraphQL
prepare ExampleTransaction1(args...) {
  # standard body as for a normal operation
}
prepare ExampleTransaction2(args...) {
  # standard body as for a normal operation
}
```

could yield

```json
{
  "transactions": {
    "ExampleTransaction1": "fdeda22b-7d56-4138-9fe8-c35a575cd466",
    "ExampleTransaction2": "958ecbff-94f1-43e3-8117-16d5fd7c4c4c"
  }
}
```
and, then:

```GraphQL
commit(transactionId: "fdeda22b-7d56-4138-9fe8-c35a575cd466")
commit(transactionId: "958ecbff-94f1-43e3-8117-16d5fd7c4c4c")
```

... yielding:
```json
{
  "transactionOutputs": {
    "fdeda22b-7d56-4138-9fe8-c35a575cd466": { ... },
    "958ecbff-94f1-43e3-8117-16d5fd7c4c4c": { ... }
  }
}
```
## Separate output area with explicit names

```GraphQL
prepare ExampleTransaction1(args...) {
  # standard body as for a normal operation
}
prepare ExampleTransaction2(args...) {
  # standard body as for a normal operation
}
```

could yield

```json
{
  "transactions": {
    "ExampleTransaction1": "fdeda22b-7d56-4138-9fe8-c35a575cd466",
    "ExampleTransaction2": "958ecbff-94f1-43e3-8117-16d5fd7c4c4c"
  }
}
```
and, then:

```GraphQL
commit xa1(transactionId: "fdeda22b-7d56-4138-9fe8-c35a575cd466")
commit xa2(transactionId: "958ecbff-94f1-43e3-8117-16d5fd7c4c4c")
```

... yielding:
```json
{
  "transactionOutputs": {
    "xa1": { ... },
    "xa2": { ... }
  }
}
```

## Root aliases

Allow aliases at the root level, possibly allowing both:

```GraphQL
exampleTransaction1: prepare(args...) {
  # standard body as for a normal operation
}
exampleTransaction2: prepare(args...) {
  # standard body as for a normal operation
}
```
... yilding

```json
{
  "data": {
    "exampleTransaction1": "fdeda22b-7d56-4138-9fe8-c35a575cd466",
    "exampleTransaction2": "958ecbff-94f1-43e3-8117-16d5fd7c4c4c"
  }
}
```

and, then:

```GraphQL
xa1: commit(transactionId: "fdeda22b-7d56-4138-9fe8-c35a575cd466")
xa2: commit(transactionId: "958ecbff-94f1-43e3-8117-16d5fd7c4c4c")
```

... yielding:
```json
{
  "data": {
    "xa1": { ... },
    "xa2": { ... }
  }
}
```

## Separate data area








# Additional out-of-box elements

## Regular expressions

Very commonly used, including here. Can help a variety of tools, including schema validators.

```GraphQL
namespace graphql.text {
  """
  Regular expression 
  """
  scalar RegularExpression

  input RegularExpressionCondition {
    """
    The pattern to match.
    """
    pattern: RegularExpression!     

    """
    Indicates whether the valid values must NOT match the specified `pattern`.
    """
    negative: Boolean! = false

    """
    Specifies the identifier of this condition, allowing tools to better indicate
    the validation failures. Note that those are ids that are meant to be definitive
    and possibly used to look up localizations.
    """
    conditionId: ID!

    """
    Optional default description.
    """
    description: String
  }
}
```

## Privacy and security directives

```GraphQL
"""
Specifies that the data is sensitive and should not be stored beyond immediate use, cached or logged.
"""
directive @graphql.secret on SCALAR, OBJECT, FIELD_DEFINITION, INPUT_FIELD_DEFINITION, INTERFACE, UNION, ENUM, INPUT_OBJECT

directive @graphql.sensitive
```

## `graphql.lettercase`

This is relevant to external handling of equivalence of values and is especially needed for aggregation tools/proxies having to match or distinguish values and compute set unions.

```GraphQL
"""
Specifies the text handling semantics for a String or ID field or a scalar definition.
"""
directive @graphql.text.meta(
    """
    Indicates whether lettercase differences affect value equality.
    """
    caseSensitive: Boolean!

    unicodeNormalization: org.unicode.NormalizationForm

    """
    Indicates whether beginning and ending whitespace affects value equality.
    """
    paddingSensitive: Boolean!

    """
    Indicates whether internal whitespace affects value equality.
    """
    internalWhitespace: Boolean!

    """
    Optionally specifies the regular expression validation - all must match.
    """
    validation: [graphql.text.RegularExpressionCondition!]
) on SCALAR, INPUT_FIELD_DEFINITION, ARGUMENT_DEFINITION, FIELD_DEFINITION

namespace org.unicode {
    enum NormalizationForm {
        """
        Canonical Decomposition
        """
        NFD

        """
        Canonical Decomposition, followed by Canonical Composition
        """
        NFC

        """
        Compatibility Decomposition
        """
        NFKD

        """
        Compatibility Decomposition, followed by Canonical Composition
        """
        NFKC
    } @specifiedBy(url: "https://www.unicode.org/reports/tr15")
} 
``` 