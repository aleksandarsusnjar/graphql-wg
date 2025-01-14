# RFC: Comprehensive Enhancements [UNFINISHED DRAFT]

> ***This is a collection of coordinated changes that work with each other.***
> While it may be possible to split those into individual parts, the aim here
> is to present them together precisely to show how they work and enhance each
> other, making sure they don't conflict.

> **References, relates to or affects the following other RFCs:**
>
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

# Safe navigation (shortcut) operators `?.` and `!.`

This is a convenience feature that can help in cases when complex types with many properties are queried by clients who only need some.
It is also leveraged to addresses the field namespacing challenges noted later. Note the relation with [RFC: Client Controlled Nullability](ClientControlledNullability.md).

The goal is to allow usages of this operator only in field selections with an explicit alias, such as:

```
  <alias> `:` <field-ref-1> ((`?.` | `!.`) <field-ref-2>)*  <opt-selection>
```

This would yield the result `<alias>` to be:

- if a specific `<field-ref-#>` in the path yields `null`:
    - `null` in case of `?.`
    - error in case of `!.`
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

> Affects: [RFC: Schema Coordinates](SchemaCoordinates.md)

To remain compatible with every possible valid implementation of prior specs, we cannot unambiguosly reuse a character that is already permitted in GraphQL identifiers within those identifiers. We could opt to split the namespace from the name in more profound ways that would not rely on this but it would yield syntax that is more cumbersome to use. Software developers using many different programming languages are already well used separating their namespaces or package names using `.` so this plays well with familiarity, readability and the learning curve.

This change introduces the `.` character as a valid part of GraphQL namespaced identifiers but with explicitly stated semantics - namespaces.

Currently, the [October 2021 GraphQL spec](https://spec.graphql.org/October2021/#sec-Names) states:

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

If `scalar`, `enum`, `union`, `interface`, `object`, `input` and directive references use unqualified identifiers then the assumed namespace is the one declared in the nearest containing (new) namespace block that contains the referenced declaration.

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

Additional rule allows us to safely define new elements in a dedicated namespace and still access it conveniently: if the above search fails to find the referenced element, one more default applies - the `graphql` namespace.

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

## Automatic (default) namespace determinism

Determining the default namespace for unqualified identifiers relies on the search across specified candidate namespaces. This is trivial to state but introduces a question which namespaces are known at any one point during parsing and what is the "content" within those namespaces at the same time.

Just using what was parsed previously would fail to account for forward references needed for cyclic relations between elements, varying order of parsing multi-file schemas and use of `extend` to modify existing definitions. Our options are to either introduce and leverage the parsing order or to define ways to address this such that the parsing order is not important. We'll try to support any order as the approach is more flexible and easier to use, though it does introduce additional parser complexity. The approach can be summed up as follows:

*Namespace defaulting for any one element will only be done after all chances of it being affected are eliminated.*

What that means in practice is that the process of translating the schema into an actionable model must be split into multiple phases:

1. Parse everything without resolving any namespaceable identifiers, qualified or not. This is simply to gather all split/spread defintions together. Keep a record of all references (again, qualified or not) that will need to be resolved.
2. Resolve `namespace` identifiers, nested and not.
3. Resolve `scalar`, `enum`, `interface`, `type`, `union`, `input` and `directive` ***declarations*** (not field or input argument type references, see next).
4. Resolve field/output and input argument type references.
5. Resolve `interface` field identifiers in interfaces that do not implement (extend) interfaces with unresolved field identifiers. Repeat this process until identifiers of all fields of all interfaces are resolved. Note that this can also be defined/done recursively - to resolve field identifiers of an interface, first ensure that all field identifiers of all interfaces it implements/extends are resolved. 
6. Resolve all `type` field identifiers.
7. Resolve all `input` field identifiers.
8. Resolve all field input argument type identifiers.

Note that relevant collision detection, validation can be done at each resolution step.

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

There have been multiple attempts to expose additional/custom schema metadata inside introspection in a variety of ways. Meanwhile, many ended up communicating the schema directly, bypassing introspection, to gain access to directives.

> This is an alternate (counter) proposal to the [RFC: Annotation Structs](AnnotationStructs.md) and, previously, [RFC: Metadata Structs](MetadataStructs.md).

The proposed approach can make the status quo valid in a clean way, by defining a new directive:

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

> Related: [RFC: GraphQL Subscriptions](Subscriptions.md) and [RFC: GraphQL Defer and Stream Directives](DeferStream.md).

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

# Allow declarations of empty types

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

# Permit interface types in concrete output

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

# Anonymous unions and in-data errors

GraphQL specification includes and approach to communicate errors already - see [7.1.2 Errors](https://spec.graphql.org/October2021/#sec-Errors). However, there are a few shortcomings of this approach that caused widespread use of an alternate way to communicate errors:

- There is no clean way to identify the kind of an error for programmatic handling, only a free-form, human-readable `message` field. Extensions could be used but are non-standard.
- The only way to attach an error to a collection data element is to have an element to attach to, to give it a distinct index for use within the `path` field. This causes issues with non-nullable element values. Either the clients have to always ignore `!` and accept anything as nullable due to an error or the schema has to abstain from marking fields that could fail as non-nullable. While both *may be* as per the spec, the utility goes down while the complexity of handling goes up.
- It is very hard for the clients to associate which fields have errors. This is especially cumbersome when distributing parts of the response to multiple different handlers whose data needs were combined into a joint GraphQL request as one can't just pass on the data chunk but has to extract and transform the errors separately.
- There is no error schema anywhere.

Aggregators of multiple GraphQL APIs face additional challenges:

- Determining whether the errors should be considered fatal even in presence of successes from other sources.
- Determining if errors are retriable.

An approach that seems to have crystalized is to use `union`s of success and error responses for failable fields. This brings the error into the data such that it is easy to distribute but introduces other issues:

- Every failable field has to have its own `union`-typed response to account for the possibility of it needing distinct error possibilities from other fields, even if they are similar.
- Even those fields that are thought of as not possible to fail may start failing in the future. The only way to preserve compatibility with that is to treat every field as possibly failing, even if it can't fail at the moment.
- This only works on complex types, not `scalar`s or `enum`s.
- Aggregators and other generic middleware cannot easily distinguish between success and error responses.

This proposal attempts to address most of the issues as follows:

1. Automatically treat every field as if it can produce errors. This is already true but ties with the other changes if explicitly stated.
2. Allow anonymous unions of mixed type kinds, i.e. `squareRoot(arg: Float!): Float! | ComplexNumber! | MathError!`. Those can be exposed as namespaced types or can be erased and every field could be treated as returning a union, perhaps of one type. Note:
    - Making a union of a scalar and a complex type is considered OK as the complex (object) value would be enclosed in the `{`...`}`.
    - A union of different kinds of scalars whose values cannot be used to distinguish the type is prohibited. For example, it is OK to make a union of `Boolean`, `Int` and `String` but not any combination of `String`, `ID` and `enum`, or `Float` and `Int`, for example. Custom scalars can be combined (or not) based on their literal JSON representations.
3. Have dedicated error `interface`, `type` and `union` definitions, marked using the preceding keyword `error`, with following requirements/restrictions:
    - Error unions can only bring together error interfaces, types and error unions.
    - Non-error unions do not have this restriction.
    - Error types can have fields of non-error types.
    - Fields of non-error types can mix error and non-error types but have to have at least one non-error type listed. In union lists the non-error types must be listed first (this is not hugely important from a functionality standpoint but may help with readability).
    - The fact that a type is an error type is introspectable via the new field in `__Type`.
4. Error values do not need to be listed in the field selection. All fields are assumed requested, including `__typename`. Fields of those errors that are of error types are also included. Fields of non-error types are not included by default and are subject to standard fragments. 
5. The requestor has to opt-in to accepting errors to be communicated this way by using `?` suffix or `?.` or `!.` joiners or explicit fragments to indicate the support for this level of GraphQL specification. Without this, the error is communicated traditionally, outside `data` but with all the error fields included within the error structure.

# Input expressions that enable powerful DSLs

A very common need to express the query *criteria* as input to GraphQL queries (and other operations) is not addressed satisfactorily by the current specification. This has caused a variety of mutually incompatible workarounds. Some of the these workarounds are:

1. Exposing criteria as a `String`, with no meaning to the GraphQL ecosystem, IDEs, aggregators, ...
   Server(s) are responsible for all parsing and making sense of the String value.
   No generic client checks are possible, type checking, syntax highlighting.
   No GraphQL comments, natural indentation alignment, context sensitive help, etc.
   Such criteria expressions cannot be generically split and distributed across multiple services
   without the "splitter + aggragator" having specific knowledge about them.
   Example: `people(criteria: "first = 'John' and last = 'Doe'") { ... }`

2. Exposing criteria as an expression tree represented as a complex, highly structured input object.
   This enables some schema-driven generic tool help but yields longer, harder to read 
   expressions that remain custom and, thus, not supportable by generic aggregators.
   Example: `people(criteria: { and: [ { first: { eq: "John" } }, { last: { eq: "Doe" }} ] }) { ... }`
    
3. Enumerating specifically needed expression patterns and exposing each individually
   via separate query or mutataion fields - "find by this", "find by that", "find by something else", etc.
   These "pre-canned" operations can offer most IDE help presently but end up to be rigid
   and hard to relate or combine by aggregators. They litter the operation namespace with what
   describes the kind of the criteria.
   Example: `peopleByFirstAndLastName(criteria: "first = 'John' and last = 'Doe'") { ... }`
   
Among those, the present workaround #2 seems most capable, yet is the least convenient to use.
The aim of this proposal is to build on that and make that approach not only more convenient
but also defined well enough such that generic tooling can interpret basic expressions and
split, refactor those as needed, without limiting the custom details that may go in. Ideally,
the end result should appear similar to what would be the value of approach #1 but with 
the structure and schema enforcement of #2. To continue the above examples, something like
the following (but not necessarily exactly that):

```GraphQL
{
  people(criteria: first == 'John' and last == 'Doe') {
    ...
  }
}
```

To do that we would need to introduce a few concepts:

1. Ability to know that a specific part of the input is an expression.

2. Ability to represent and parse those expressions in/from a more human-friendly format,
   which has been practically nailed down to the infix operator notation.

3. As noted in #2, operators. A basic built-in set is needed, with the ability to 
   chose which ones are supported and add custom.

4. Ability to refer to contextual fields.

5. Ability to control the context, say if downcasts are required or simply using
   the type as a criteria value. This also means that we need to be able to reference
   a specific subset of allowable types as a value of an input field, operand or
   argument.

6. Name candidate objects and be able to reference them by those names.

At this time I'd like to point out that, although this can certainly help data-driven
APIs and services that are close to the database, that is not the focus of this proposal.
If it were, we might as well try to codify some existing standard. 

That is not the goal. The goal is to help clients and servers communicate criteria
at the evel of API-exposed "business model" which is assumed to be a higher level
abstraction and a transformation of whatever and however the data is stored, if it is at all.

An interesting possibility is that we could  drive this so that input expressions
can be expressed two ways - either in whatever new form we come up with or as an equivalent
input object structure that could aid some clients implementations and/or the transition.

Consider the following definition:

```GraphQL
input Expression @oneOf {      # Leverages upcoming directive not yet in standard
  all:        [Expression!]    # N-ary "and" operator
  any:        [Expression!]    # N-ary "or" operator
  none:       [Expression!]    # N-ary "not" operator

  eq:         [Expression]     # N-ary "==" operator
  ascending:  [Expression!]    # N-ary "<" operator
  descending: [Expression!]    # N-ary ">" operator
  # etc

  # Terminal values, allowing for some artistic freedom (i.e. not to spec presently)

  literal:    __Any            # Literal value
  path:       [__FieldName!]!  # Reference to a field relative to present scope
  typeRef:    __TypeRef!       # Reference to a type
  namedExpr:  ExpressionName!  # Reference to a named expression, see below

  # Ability to declare named expressions:
  using:  UsingExpression!
}

input UsingExpression {
  namedExpressions: [NamedExpression!]!
  expression: Expression!
}

input NamedExpression {
  expression: Expression!
  name: ExpressionName!
}

scalar __TypeRef
scalar __FieldName
scalar ExpressionName
```

This looks like it could express many expressions using the workaround approach #2 but with
all its verbosity and inconvenience. It also lacks some abilitites - e.g. how can a tool
"know" which exact/concrete types are permitted either for, say, `typeRef` or `path`
and what kind of type agreement needs to exist for nary operators?

Step by step...

## Prefix and infix notation for familiarity

Let's introduce the following:

```GraphQL
namespace graphql {
  scalar InfixableOperator

  enum OperatorArity {
    UNARY
    BINARY # optional?
    NARY
  }

  enum OperatorAssociativity { # Optional, needed for binary only.
    LEFT
    RIGHT
  }

  directive @function(                  # prefix operator(operand, operand, ...)
    operandTypes:  [TypeRef!]
    yields:        TypeRef!
  )

  directive @operator(                   # infix operand op operand op operand ... 
    arity:         OperatorArity!
    op:            [InfixableOperator!]
    operandTypes:  [TypeRef!]            # optional, no value => operand types must be equal
    yields:        TypeRef!
    precedence:    Float!
    associativity: OperatorAssociativity # matters only to binary
  ) on INPUT_FIELD_DEFINITION

  directive @expression(
    contextType:   TypeRef!
    yields:        TypeRef!
  ) on INPUT_OBJECT

  directive @predicate( # shorthand for Boolean-yielding @expression
    contextType:   TypeRef!
  ) on INPUT_OBJECT
```

In the above ideally `@expression` and `@predicate` would imply `@oneOf` and would not require
it too to be specified. That, however, works only if they come out in the same version of the
GraphQL spec.

Given that, let's rewrite the top part of the initial example:

```GraphQL
input PersonPredicate @predicate(contextType: Person) {
  all:        [PersonPredicate!]  @operator(arity: NARY,  op: "and" operandTypes: Boolean yields: Boolean precedence: 2)
  any:        [PersonPredicate!]  @operator(arity: NARY,  op: "any" operandTypes: Boolean yields: Boolean precedence: 1)
  not:         PersonPredicate    @operator(arity: UNARY, op: "not" operandTypes: Boolean yields: Boolean precedence: 3)

  eq:         [Expression]        @operator(arity: NARY,  op: "=="  yields: Boolean precedence: 4)
  ascending:  [Expression!]       @operator(arity: NARY,  op: "<"   yields: Boolean precedence: 4)
  descending: [Expression!]       @operator(arity: NARY,  op: ">"   yields: Boolean precedence: 4)
  # etc
  ...
```

Given careful restrictions on what can constitute allowable `InfixableOperator`
values, parsers can dynamically configure themselves to accept them.
It would be nice to leverage standard names for most commonly used operators such
as those noted here. Custom extensions (to be discussed later) could be forced to 
use specialized decorations, akin to prefixing variable names with `$`.

> TO DO: Exact custom operator approach to be decided.

A supporting parser that knows it can expect an expression needs to be ready for the following 
situations:

1. The expression does not begin with `{`. That would indicate the legacy, expression tree approach.

2. The next token can and will usually be a term that may be followed by a joining infix operator
   suggesting the continuation of the expression.

3. Ideally it can expect and support the use of parentheses to disambiguate, simplify
   and enable more complex expressions.

## What are those types in there?

In the example above, notice here is that the types referred here, such as `Person` and `Boolean`
are not input types. They are (possibly interim) results of expression evaluations. That also
makes them not necessarily output types either. They demand a new type namespace.

Just like for input and output types, it seems feasible to share the scalar and enum definitions
and treat them as contextual types too. If specific scalar/enum types are needed only for
expression results, they can just be ignored and unreferenced from "proper" input/output types
and/or specifically marked so.

Complex expression result types are, however, required to be able to reference and navigate
their fields and keep track of contextual types within expressions. A hierarchy will very likely
be needed as these often align very well with actual output types but may have somewhat different
set of abilities - e.g. just a subset of fields. Perhaps a safer approach is to not rely on this,
the proposed approach is to introduce a dedicated keyword for these types, such as `context`.
For example:

```GraphQL
"""
Defines an expression context type "Person"
"""
context Person {
  first: String
  last:  String
}
```

> TO DO Discuss sharing output and expression types with directive-annotated exclusions/differences.

## Boilerplate reduction and standardization

To avoid re-declaring much of the boilerplate needed for the standard complex expressions,
the previous example of declaring the `context Person { ... }" context type can have more 
than one effect. Namely it can also have a sideffect of defining the `Person.Expression`
type. This would be implicit for all scalars and enums, so there should exist 
`Boolean.Expression`, `Int.Expression`, `Float.Expression`, `ID.Expression`, `String.Expression` 
and `<enum>.Expression` types as well, each yielding the same type. Additionally and,
perhaps more importantly, each would also cause a corresponding `<type>.Predicate` expression type
that yields a `Boolean`.

 Particularly, defining:

```GraphQL
context Person {
  first: String
  last:  String
}
```

... would implicitly also define all of the following:

```GraphQL
input Person.Predicate @predicate(contextType: Person) {
  all:        [PersonPredicate!]  @operator(arity: NARY,  op: "and" operandTypes: Boolean yields: Boolean precedence: 2)
  any:        [PersonPredicate!]  @operator(arity: NARY,  op: "any" operandTypes: Boolean yields: Boolean precedence: 1)
  not:         PersonPredicate    @operator(arity: UNARY, op: "not" operandTypes: Boolean yields: Boolean precedence: 3)
  ...
}
input Person.Expression @expression(contextType: Person) {
  # No all/any/not here, except for Boolean

  ...
}
```

At this point, if not before, the question arises as to which other operators not included in
the above example are allowed, should be added and how? The separation between standard
and custom operators may be leveraged here to avoid boilerplate while helping with the definition.
For example, by allowing context types to have argument lists, and defining the following:

```GraphQL
namespace graphql {
  enum ExpressionSupport {
    NONE              # Don't even mention it.
    TERM_ONLY         # Just a single term
    SINGLE_OPERATOR   # No and/or combos
    COMPLEX           # The whole enchilada
  }
}

we could express the following:

```GraphQL
context Person(
  expressions: NONE   # No need for Person.Expression
  predictes: COMPLEX  # Bring it on for Person.Predicate
  equatable: true     # Yes, we can do == and != for Persons
  orderable: false    # No, we don't want <, <=, =>, >
  ...
) {
  first: String
  last:  String
}
```

... we could control the inclusion/support for standard GraphQL expression features.

## Custom operators

As `<type>.Expression` and `<type>.Predicate` types are automatically created,
we don't expect a complete definition, only extensions to these types. For example:

```GraphQL
extend input Person.Predicate {
  registeredBefore: [Expression!] @operator(
    arity: BINARY
    op: "~before"
    operandTypes: [Person, Timestamp]
    yields: Boolean
    precedence: 3.5
  )
}
```

## TO DO: Term constrants

```GraphQL
  literal:    __Any            # Literal value
  path:       [__FieldName!]!  # Reference to a field relative to present scope
  typeRef:    __TypeRef!       # Reference to a type
  namedExpr:  ExpressionName!  # Reference to a named expression, see below

  # Ability to declare named expressions:
  using:  UsingExpression!
}
```