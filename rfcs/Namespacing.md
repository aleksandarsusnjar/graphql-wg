# RFC: Namespace support

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar)

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**See also:**
- [RFC: GraphQL Composite Schemas](CompositeSchemas.md)
- [RFC: Transactions](Transactions.md)

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

A lone double-underscore `__` is specially treated as a reference to the root namespace.

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

## Compatibility Considerations

### Legacy client + new server

Legacy clients do not use the new feature in their queries.
The exact comprehension (meaning) of the legacy query remains unchanged.

### New client + legacy server

For clients that do not have runtime discovery and rely on queries written before runtime there 
are no issues as they would be written specifically for a version of the server and would
use the correct feature set.

Clients that need to be able to work with multiple different server versions need a way 
to find out whether the server (version) supports this feature or not. This can be done
with a custom-made query field.

Generic clients, such as IDEs must rely on standard server feature discovery, 
perhaps as in [RFC: Feature Discovery](FeatureDiscovery.md).

Additionally, such clients can resort to probing / testing with a query such as:

```GraphQL
query NamespaceSupportDiscovery {
    __.graphql.Query { __typename }
}
```

Such a query would fail if introspection is disabled but, at that point, it is questionable
what such tools can do and how they would offset the lack of schema discovery. 