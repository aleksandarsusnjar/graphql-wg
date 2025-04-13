# RFC: Safe navigation (shortcut) and refactoring operators

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar)

**See also / part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

## Problem Statement

GraphQL presently enables clients to tailor the response structure by:

- Selecting only a subset of the schema available within any composite output type scope.
- (Re)name output fields by via aliases.
- Include multiple instances of otherwise same field, potentially with different input arguments, by using different aliases.

These allow some schema remapping but does not allow them to "flatten" or "simplify" deeper structures.
Take the following schema example:

```GraphQL
type PersonName {
    given: String!
    family: String
    # ...
}

type Person {
    id: ID!
    name: PersonName!
    email: Email    
    # ...
}

type Document {
    id: ID!
    title: String!
    authors: [Person!]
    # ...
}

type Product {
    id: ID!
    name: String!
    brochure: Document
    # ...
}

type Query {
    products: [Product!]
    # ...
}
```

To get the products' brochure's author's given names one would have to execute something like:

```GraphQL
query {
    products {
        brochure {
            authors {
                name {
                    given
                }
            }
        }
    }
}
```

Navigating this structure requires additional code in the client, either to use it directly or 
transform it into another form that is usable by the other/existing client logic.

## Proposed Solution

Introduce [safe navigation operators](https://en.wikipedia.org/wiki/Safe_navigation_operator) (a.k.a. optional chaning, null-conditional and/or null-propagation) suitable for use within GraphQL query aliases.

This is also leveraged to addresses the field namespacing challenges referenced in other parts of the [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)
Also note the relation with [RFC: Client Controlled Nullability](ClientControlledNullability.md).

### Specification change

Change the [Fields](https://github.com/graphql/graphql-spec/blob/main/spec/Section%202%20--%20Language.md#fields) section from:

> Field : Alias? Name Arguments? Directives? SelectionSet

to:

> Field : Alias? FieldPath SelectionSet

and introduce the following:

> FieldAccess: Name Arguments? Directives? 
>
> FieldPath: FieldAccess ( SafeNavOperator FieldAccess )* DistinctScalarSuffix?
>
> SafeNavOperator: '?.' | '!.' | '*.' | '+.'
>
> DistinctScalarSuffix: '*' | '+'

... with the following meaning of each operator:

- `?.` - A variant of the [Safe navigation operator](https://en.wikipedia.org/wiki/Safe_navigation_operator) that expands/propagates collections, more on that below.
- `!.` - alternate version that yields an error in `null` is encountered. This is usually just a `.` in other languages but this proposal maintains symmetry with GraphQL's use of `?` and `!` to explicitly denote nullability. Additionally by not relying on `.` here, that token remains available for other uses as a simple part of a field name, such as used in the [Namespace support`](Namespacing.md) proposal.
- `*.` - Same as `?.` when navigating *from* single-value fields and contexts, otherwise requests distinct / unique values only according to the identity as defined by the server. 
- `+.` - a `!.` equivalent for `*.`, requiring at least one element (e.g. in a list), not just non-null and yielding an error otherwise.

#### Examples

Given the schema example from the beginning, the query example above could be rewritten in multiple ways as follows, to achieve different goals:


```GraphQL
query {
    # Given names of all authors of all brochures of all products.
    # There may be many duplicates as same people may have authored
    # multiple brochures, each of wwhich may apply to multiple products.
    # Products without brochures or brochures without authors are ignored / skipped.
    x1: products?.brochure?.authors?.name?.given

    # Same as above but make sure each brochure is considered only once.
    # Assuming any one person can appear at most once as an author of any
    # one brochure, but it is possible that multiple different people
    # have equal given names and, even full names.
    x2: products?.brochure*.authors?.name?.given

    # Same as above but this time we're forcing the uniqueness on authors
    # (people), likely yielding the same results, perhaps with different efficiency.
    x3: products?.brochure?.authors*.name?.given

    # Similar to the above but erroring out if missing authors or their names
    # and returning only given names of otherwise distinct full names.
    x4: products?.brochure*.authors+.name+.given

    # Similar to the above but ensuring we have given names
    x5: products?.brochure*.authors+.name+.given+

    # Compound output
    x6: products?.brochure*.authors+.name { given family }

    # Another refactoring:
    x7: products?.brochure* {
        brochure_id: id
        authorsNames: authors+.name { 
            first: given 
            last: family
        }
    }
}
```

Assuming the following data:

- Product id=1: Cup
    - Brochure id=1:
        - Author id=1: Ann Smith
        - Author id=2: Ann Doe
- Product id=2: Mug
    - Brochure id=1: shared with product id=1
- Product id=3: Bowl
    - Brochure id=2:
        - Author id=1: Ann Smith (same as in brochure id=1)
        - Author id=3: Ann Smith (same name but actually different person)
- Product id=4: Plate
    - Brochure id=3
        - Author id=4: Bob Smith

Joint JSON output would be:

```json
{
  "data": {
    "x1": [ 
        "Ann", // product 1 -> brochure 1 -> author 1: Ann Smith (#1)
        "Ann", // product 1 -> brochure 1 -> author 2: Ann Doe
        "Ann", // product 2 -> brochure 1 -> author 1: Ann Smith (#1)
        "Ann", // product 2 -> brochure 1 -> author 2: Ann Doe
        "Ann", // product 3 -> brochure 2 -> author 1: Ann Smith (#1)
        "Ann", // product 3 -> brochure 2 -> author 3: Ann Smith (#2)
        "Bob"  // product 4 -> brochure 4 -> author 4: Bob Smith
    ],    
    "x2": [ 
        "Ann", // product 1 -> brochure 1 -> author 1: Ann Smith (#1)
        "Ann", // product 1 -> brochure 1 -> author 2: Ann Doe
        // Duplication of brochure 1 by product 2 skipped
        "Ann", // product 3 -> brochure 2 -> author 1: Ann Smith (#1)
        "Ann", // product 3 -> brochure 2 -> author 3: Ann Smith (#2)
        "Bob"  // product 4 -> brochure 4 -> author 4: Bob Smith
    ],
    "x3": [ 
        "Ann", // product 1 -> brochure 1 -> author 1: Ann Smith (#1)
        "Ann", // product 1 -> brochure 1 -> author 2: Ann Doe
        // Duplication of author id=1 skipped
        // Duplication of author id=2 skipped
        // Duplication of author id=1 skipped
        "Ann", // product 3 -> brochure 2 -> author 3: Ann Smith (#2)
        "Bob"  // product 4 -> brochure 4 -> author 4: Bob Smith
    ],
    "x4": [ 
        "Ann", // product 1 -> brochure 1 -> author 1: Ann Smith (#1)
        "Ann", // product 1 -> brochure 1 -> author 2: Ann Doe
        // Duplication of brochure 1 by product 2 skipped
        // Duplication of author id=1 skipped
        "Ann", // product 3 -> brochure 2 -> author 3: Ann Smith (#2)
        "Bob"  // product 4 -> brochure 4 -> author 4: Bob Smith
    ],
    "x5": [ 
        "Ann", // product 1 -> brochure 1 -> author 1: Ann Smith (#1)
        // Duplication of "Ann" skipped
        // Duplication of brochure 1 by product 2 skipped
        // Duplication of author id=1 skipped
        // Duplication of "Ann" skipped
        "Bob"  // product 4 -> brochure 4 -> author 4: Bob Smith
    ],
    "x6": [
        { "first": "Ann", "last": "Smith" }, // Ann Smith #1
        { "first": "Ann", "last": "Doe"   },
        // Duplication of brochure 1 by product 2 skipped
        // Duplication of author id=1 skipped
        { "first": "Ann", "last": "Smith" }, // Ann Smith #2
        { "first": "Bob", "last": "Smith" }
    ],
    "x7": [
        {
            "brochure_id": 1,
            "authorNames": [
                { "first": "Ann", "last": "Smith" },
                { "first": "Ann", "last": "Doe"   }
            ]

        },
        // Duplication of brochure 1 by product 2 skipped
        {
            "brochure_id": 2,
            "authorNames": [
                { "first": "Ann", "last": "Smith" },
                { "first": "Ann", "last": "Smith" }
            ]

        },
        {
            "brochure_id": 3,
            "authorNames": [
                { "first": "Bob", "last": "Smith" }
            ]

        }
    ]
  }
}
```

Additionally, `x4`-`x7` would have produced errors if corresponding fields were `null` or empty.

## Trackback

For additional power, each complex output type context could be dynamically enriched with a
special "breadcrumbs trackback" `__back` field identifying the incoming paths. For example:

```GraphQL
query {
    x8: products?.brochure*.authors+ { 
        authorId: id
        firstName: name?.given 
        lastName: name?.family
        bibliography: __back* { # brochures the author authored, as encountered during descent
            brochureId: id
            brochureTitle: title
            relevantProducts: __back {
                id
                name
            }
        }
        productsWroteAbout: __back*.__back*.name
    }
}
```

... yielding:

```json
{
  "data": {
    "x8": [
        { 
            "authorId": 1,
            "firstName": "Ann", 
            "lastName": "Smith",
            "bibliography": [
                {
                    "brochureId": 1,
                    "brochureTitle": "...",
                    "relevantProducts": [
                        { "id": 1, "name": "Cup" }
                        { "id": 2, "name": "Mug" }
                    ]
                },
                {
                    "brochureId": 2,
                    "brochureTitle": "...",
                    "relevantProducts": [
                        { "id": 3, "name": "Bowl" }
                    ]
                }
            ],
            "productsWroteAbout": [ "Cup", "Mug", "Bowl" ]
        }, 
        {
            "authorId": 2,
            "firstName": "Ann",
            "lastName": "Doe",
            "bibliography": [
                {
                    "brochureId": 1,
                    "brochureTitle": "...",
                    "relevantProducts": [
                        { "id": 1, "name": "Cup" }
                        { "id": 2, "name": "Mug" }
                    ]
                }
            ],
            "productsWroteAbout": [ "Cup", "Mug" ]
        },
        // Duplication of brochure 1 by product 2 skipped
        // Duplication of author id=1 skipped
        {
            "authorId": 3,
            "firstName": "Ann",
            "lastName": "Smith",
            "bibliography": [
                {
                    "brochureId": 2,
                    "brochureTitle": "...",
                    "relevantProducts": [
                        { "id": 3, "name": "Bowl" }
                    ]
                }
            ],
            "productsWroteAbout": [ "Bowl" ]
        },
        {
            "authorId": 4,
            "firstName": "Bob",
            "lastName": "Smith",
            "bibliography": [
                {
                    "brochureId": 3,
                    "brochureTitle": "...",
                    "relevantProducts": [
                        { "id": 4, "name": "Plate" }
                    ]
                }
            ],
            "productsWroteAbout": [ "Plate" ]
        }
    ]
  }
}
```

The same can apply to scalar which would necessitate field selection as with compound types.
That, in turn, would necessitate a way to reference the value itself, perhaps with `__value`.

Example of this query-time promotion of scalar fields to compound:

```GraphQL
query {
    x9: products?.brochure*.authors+.name?.given { # 'given' is a scalar field        
        authorId: __back!.__back!.id
        firstName: __value
    }
}
```

yielding:

```json
{
  "data": {
    "x9": [ 
        { "authorId": 1, "firstName": "Ann" }, // product 1 -> brochure 1 -> author 1: Ann Smith (#1)
        { "authorId": 2, "firstName": "Ann" }, // product 1 -> brochure 1 -> author 2: Ann Doe
        // Duplication of brochure 1 by product 2 skipped
        // Duplication of author id=1 skipped
        { "authorId": 3, "firstName": "Ann" }, // product 3 -> brochure 2 -> author 3: Ann Smith (#2)
        { "authorId": 4, "firstName": "Bob" }  // product 4 -> brochure 4 -> author 4: Bob Smith
    ]
  }
}
```


This, however, may not be necessary as the similar can be accomplished by only navigating to the
immediate container:

```GraphQL
query {
    x9: products?.brochure*.authors+.name { 
        authorId: __back!.id
        firstName: given
    }
}
```

or:

```GraphQL
query {
    x9: products?.brochure*.authors+ { 
        authorId: id
        firstName: name?.given
    }
}
```


## Compatibility Considerations

### Legacy client + new server

Legacy clients do not use the new feature in their queries.
The exact comprehension (meaning) of the legacy query remains unchanged.

### New client + legacy server

For clients do not have runtime discovery and rely on queries written before runtime there 
are no issues as they would be written specifically for a version of the server and would
use the correct feature set.

Clients that need to be able to work with multiple different server versions need a way 
to find out whether the server (version) supports this feature or not. This can be done
with a custom-made query field.

Generic clients, such as IDEs must rely on standard server feature discovery, 
perhaps as in [RFC: Feature Discovery](FeatureDiscovery.md).

Additionally, such clients can resort to probing / testing with a query such as:

```GraphQL
query SafeNavigationOperatorSupportQuery {
    schemaDescription: __schema?.description
}
```

Such a query would fail if introspection is disabled but, at that point, it is questionable
what such tools can do and how they would offset the lack of schema discovery. 

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
