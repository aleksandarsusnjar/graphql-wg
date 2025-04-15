# RFC: Anonymous types

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Relied upon by:**
- [RFC: In-data errors](InDataErrors.md)
- [RFC: Input unions](InputUnions.md)
- [RFC: Type References](TypeReferences.md)
- [RFC: Empty Types](EmptyTypes.md)

**May help with:**
- [RFC: GraphQL Composite Schemas](CompositeSchemas.md)
- [RFC: Namespace support](Namespacing.md)

## Problem statement

Schemas get verbose due to the need to explictly name and declare a (single) type
for every input or output field, or input or directive argument. Other than 
readability issues this also augments the problems navigating (introspecting)
very large schemas, such as those composed out of many smaller schemas.

Additionally, multidimensional collections, most commonly tables (2D) presently
require explicit named declarations of row type - no tuples support.

## Proposal

### Input arguments (fields and/or directives)

Notice the following declaration:

```GraphQL
extend type Query {
    example(a: Int!, b: Float!, c: String!): Something
}
```

In the above example, `a: Int!, b: Float!, c: String!` could be viewed as an anonymous
type. On the server, input arguments can be commonly passed together in a single 
dedicated object containing all of them and there exist recommendations to even
declare a single `input` argument only, for a variety of reasons. 

These are enveloped within `(`...`)` right after the field name:

```
(a: Int!, b: Float!, c: String!)
```

We can exploit that syntax to come up with something similar for anonymous input 
types.


#### Alternative 1: argument's arguments (not recommended)

This alternative makes input arguments have their own input arguments.

```GraphQL
extend type Query {
    example(name(given: String!, family: String!)!, email: EmailAddress!): Person
    # notice    ^                               ^^
}
```

Notice the placenet of `!` after the `)`.

Not recommended because:

- Cannot be used for list/array/collection element types.

- The implied corresponding/equivalent request syntax/structure would not be compatible
  if the anonymous type is, later, promoted to a named type, or vice versa.

- There is no clean way to specify the type kind for/if/when that is required, such
  as for `union` in [RFC: Input unions](InputUnions.md).

#### Alternative 2: nested arguments (not recommended)

This alternative addresses the consistency with how the requests would pass data with
both named and anonymous input types.

```GraphQL
    example(name: (given: String!, family: String!)!, ...): ...
    # notice      ^                               ^^
```

Notice the placenet of `!` after the `)`.

List element types can be supported as follows:

```GraphQL
    [(given: String!, family: String!)!]
```

Not recommended because:

- There is no clean way to specify the type kind for/if/when that is required, such
  as for `union` in [RFC: Input unions](InputUnions.md).


#### Alternative 3: nested nameless type declaration (recommended)

```GraphQL
    example(name: input { given: String!, family: String! }!, ...): ...
    # notice      ^^^^^^^                                 ^^
```

Notice the placenet of `!` after the `)`.

List element types can be supported as follows:

```GraphQL
    [input { given: String!, family: String! }!]
```

Also note that, even if we allow `input` to be a type name (scalar, enum or input), 
the fact that it is followed by `{` disambiguates this case.

There is a clean way to specify the type kind for/if/when that is required, such
as for `union` in [RFC: Input unions](InputUnions.md) (not a part of this proposal),
just an illustration):

```GraphQL
    power(name: union Int! | Float!, degree: union Int! | Float!): Float!
    # notice    ^^^^^    ^ ^      ^          ^^^^^    ^ ^      ^
```

Existing way of supplying default argument values can work for non-union types:

```GraphQL
    example(
        name: input { 
            given: String! = "Joe", 
            family: String! = "Doe"
        }!, 
        email: EmailAddress! = "jdoe@does.com"
    ): Person
```

... but that wouldn't work for and be consistent with unions, so the following is recommended
as an additional way:

```GraphQL
    example(
        name: input { given: String!, family: String! }! = {            
            given: "Joe", 
            family: "Doe"
        }, 
        email: EmailAddress! = "jdoe@does.com"
    ): Person
```

An illustration as to how this can work with `union` in [RFC: Input unions](InputUnions.md)
(not a part of this proposal):


```GraphQL
    power(name: union Int! | Float!, degree: union Int! | Float! = Int: 2): Float!
    # notice    ^^^^^    ^ ^      ^          ^^^^^    ^ ^      ^ ^^^^^^^^

    # or

    power(name: union Int! | Float!, degree: union Int! | Float! = { Int: 2 }): Float!
    # notice    ^^^^^    ^ ^      ^          ^^^^^    ^ ^        ^ ^^^^^^^^^^
```

The above example makes use of [RFC: Type References](TypeReferences.md) too.

#### What about anonymous input interfaces?

As these types are anonymous, they cannot be referenced for further extension or
implementation. Thus, even if/when input interfaces are added, it would not really
matter - these nested input types could safely be considered "concrete" types.


### Output field types

Equivalent of the "alternative 3" for input arguments works for output types too 
and maintains syntactic consistency:

```GraphQL
    name(personId: ID!): type { given: String, family: String }
    customer(id: ID!): union Person | Business
```

#### Union ambiguity

A standard, named union of named unions can safely resolve and "deduplicate"
type options. The same applies to an anonymous union(s) of named types
(of any kind). Thus, any hierarchy of either/both named and anonymous unions
is fine as long as they don't reference anonymous (unnamed) non-union tyes
(i.e. the `type` or `interface` kind).

For example, in the following:

```GraphQL
union Number = Int | Float
scalar Long

extend type Query {
   someNumber: union Number | union Int | Float | Long
}
```

... it is clear that `someNumber` can either be `Int`, `Float` or `Long`
but the comprehension is a bit clumsy as there are multiple ways to interpret
the grouping needed to form the graph of the schema. There are three possible
interpretations:

**Interpretation 1**

```
    someNumber +-- anonymous -+- Number   -+- Int
                              |           -+- Float
                              |
                              + anonynous -+- Int
                                           +- Float
                                           +- Long
```

**Interpretation 2**

```
    someNumber +-- anonymous -+- Number    -+- Int
                              |             +- Float
                              |
                              +- anonynous -+- Int
                              |             +- Float
                              |
                              +- Long
```

**Interpretation 3**

```
    someNumber +-- anonymous -+- Number    -+- Int
                              |            -+- Float
                              |
                              +- anonynous -+- Int
                              |                              
                              +- Float
                              +- Long
```

As anonymous unions help with convenience and brevity and nesting anonymous
unions inside anonymous unions both violates this goal and isn't functionally
necessary - we could just omit the second `union` keyword, 
**using anonymous unions inside other unions (named or anonymous)
is prohibited in this proposal**.

#### Fragment ambiguity

Fragments cannot "lock on to" (state type conditions) on unnamed types.
This is not an issue for fields that have a singular non-union, i.e. `type`
or `interface` output type defined as the selection can be done on the
field itself and individual fields within the output.

However, it is impossible to distinguish between unnamed non-union 
choices within a union


Fragments cannot "name" (reference) nameless anonymous types. Example:

```GraphQL
type Person { ... }
union Customer = Person | type { companyName: String! }

extend type Query {
    customer(id: ID!): Customer
}
```

would cause issues with query such as:

```GraphQL
query GetBusinessCustomer {
    customer(id: 123) {
        ... on ???? { # What would be the condition for the non-Person customer?
            companyName
        }
    }
}
```

As such, **using anonymous output types of any kind is prohibited in 
unions of any kind**.

### Interface implementation

Consider the following example:

```GraphQL
interface Foo {
    foo(x: input { a: Int!, b: Float! }!): union Int | Float
}

type Bar implements Foo {
    foo(x: ???): ???
    ...
}
```

What should the `???` placeholders become?

#### Alternative 0: Prohibit

Prohibit the use of anonymous types in interfaces to avoid this issue, for now.

#### Alternative 1: Repeat, verbatim (recommended for now)

This is consistent with how GraphQL requires implementations to work, but is verbose.

```GraphQL
interface Foo {
    foo(x: input { a: Int!, b: Float! }!): union Int | Float
}

type Bar implements Foo {
    foo(x: input { a: Int!, b: Float! }!): union Int | Float
    ...
}
```

The benefit of this approach is that ensures implementors to be notified if/when
upstream interface declarations change - static build-time check.

#### Alternative 2: Introduce special references to the super-declaration

Less verbose but without the change "notification" benefit of the alternative 1.

```GraphQL
interface Foo {
    foo(x: input { a: Int!, b: Float! }!): union Int | Float
}

type Bar implements Foo {
    foo(x: __inherit): __inherit
    ...
}
```

GraphQL presently doesn't support field overloading so fields are keyed by their
names, without their input arguments. If this were ever to change, we would not
be able to identify the overloaded variant.

The prefix `__` in `__inherit` is here to prevent a conflict with a possible
custom type named `inherit`.

This version allows permitted variations:

- Anonymous input `union` arguments could accept more types than the superimplementation.
- Anonymous output `union` type could be narrowed down to a smaller selection.
- Anonymous input `type` arguments could have additional optional fields (either nullable or having a default value)
- Anonymous output `type` and/or `interface` type could contain additional fields.


#### Alternative 3: Explictly inherit the selected field(s)

Even less verbose but without the change "notification" benefit of the alternative 1 
or finer-grain control of other prior alternatives.

**Subvariant (a)**

```GraphQL
type Bar implements Foo(foo) {
    ... # could be empty if foo is the only field
}
```

**Subvariant (b)**

```GraphQL
type Bar implements Foo @graphql.inherit(fields=["foo"]) {
    ... # could be empty if foo is the only field
}
```

**Subvariant (c)**

```GraphQL
type Bar implements Foo {
    foo @graphql.inherit
}
```

#### Alternative 4: Inherit all fields

Least verbose but without the change "notification" benefit of the alternative 1 
or finer-grain control of other prior alternatives.

**Subvariant (a)**

```GraphQL
type Bar inherits Foo {
    ... # could be empty if foo is the only field
}
```

**Subvariant (b)**

```GraphQL
type Bar implements Foo @graphql.inheritAll {
    ... # could be empty if foo is the only field
}
```

**Subvariant (c)**

```GraphQL
type Bar implements Foo+ {
    ... # could be empty if foo is the only field
}
```

#### Alternative 5: Combination

Alternatives 1, 2, 3 and 4 above are not exclusive. A combination can be supported
together. 

### Introspection

As anonymous types aren't named, they cannot be looked up by name.
Due to implementation/inheritance, they cannot necessarily state a
a single place of declaration - there may be multiple. However, they 
can be referenced by arguments/fields.

To support this the introspection schema needs to be updated with additional
`__TypeKind` enum possibilities:

```GraphQL
enum __TypeKind {
  SCALAR
  OBJECT
  INTERFACE
  UNION
  ENUM
  INPUT_OBJECT
  LIST
  NON_NULL
  ANONYMOUS_OBJECT       # new
  ANONYMOUS_INTERFACE    # new, optional
  ANONYMOUS_UNION        # new
  ANONYMOUS_INPUT_OBJECT # new
  
  # with polymorphic inputs resolved:
  INPUT_INTERFACE        # new  
  ANONYMOUS_INTERFACE    # new
  INPUT_UNION            # new
  ANONYMOUS_INPUT_UNION  # new
}
```

The `__Type` type appears to already have a nullable  `name: String` field
and would need to keep it that way.

## Addendum: tuples, tables and key-value pairs example

```GraphQL
   temperatureLog: [type { recordedAt: DateTime!, temperature: Float! }!]
   keyValuePairs:  [type { key: String!, value: String! }!]
```