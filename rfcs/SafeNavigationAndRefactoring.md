# RFC: Safe navigation (shortcut) and refactoring operators

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

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
> FieldPath: FieldAccess ( SafeNavInfixOperator FieldAccess )* FieldPathSuffixOperator?
>
> SafeNavInfixOperator: '?.' | '!.' | '*.' | '+.'
>
> FieldPathSuffixOperator: '*' | '+' | '??' | '?*' | '**'

Terminators and operat

... with the following meaning of each operator characters:

#### The meaning of the first character

- `?` - all incoming non-null contexts with null propagation
- `!` - all incoming non-null context with null yielding an error
- `*` - distinct incoming non-null context with null propagation
- `+` - distinct incoming non-null context with null yielding an error

#### The meaning of the second character

Infix operators:

- `.` - navigation

Suffix operators:

- `?` - non-null existence check, yields a `Boolean` value `true` if at least one non-null
        value would be returned if this (second) character is not present, otherwise `false`.
- `*` - non-null count, yields a `Int` countof non-null count of non-null
        values that would be returned if this character were not present.

#### Full infix operator meanings        

- `?.` - A variant of the [Safe navigation operator](https://en.wikipedia.org/wiki/Safe_navigation_operator) that expands/propagates collections, more on that below.
- `!.` - alternate version that yields an error in `null` is encountered. This is usually just a `.` in other languages but this proposal maintains symmetry with GraphQL's use of `?` and `!` to explicitly denote nullability. Additionally by not relying on `.` here, that token remains available for other uses as a simple part of a field name, such as used in the [Namespace support`](Namespacing.md) proposal.
- `*.` - Same as `?.` when navigating *from* single-value fields and contexts, otherwise requests distinct / unique values only according to the identity as defined by the server. 
- `+.` - a `!.` equivalent for `*.`, requiring at least one element (e.g. in a list), not just non-null and yielding an error otherwise.

#### Full suffix operator meanings

- `??` - Returns a `Boolean` value of `true` if the the path without this 
  suffix operator would otherwise **have** at least one non-null value, otherwise
  `false`.

- `?*` - Returns an `Int` **count** of non-null values the path without this
   suffix operator would otherwise yield.

- `**` - Returns an `Int` **count** of non-null **distinct** values the path without this
   suffix operator would otherwise yield.

The following operators are excluded:

- `*?` - would yield the same result as `?*`
- `+?` - would either yield an error or `true`, not considered useful.
- `+*` - would either yield an error in cases `**` would return `0`, otherwise equivalent.

Suffix operators reduce the need for special existence checks and counting 
fields, standardize the approach and can reduce both communication size and
processing times (server load) when clients just need counts.

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

    brochureAuthorsExist:   products?.brochure?.authors??
    brochureAuthorsCount:   products?.brochure?.authors** # distinct
    distinctGivenNameCount: products?.brochure?.authors?.name?.given**
    x1ResultCount:          products?.brochure?.authors?.name?.given?*

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
    ],
    "brochureAuthorsExist":   true, // Yes, some brochures have autohors
    "brochureAuthorsCount":   4,    // Ann Smith #1, Ann Smith #2, Ann Doe and Bob Smith
    "distinctGivenNameCount": 2,    // Ann and Bob
    "x1ResultCount":          7     // Ann, Ann, Ann, Ann, Ann, Ann and Bob
  }
}
```

Additionally, `x4`-`x7` would have produced errors if corresponding fields were `null` or empty.

### Need for aliases

Aliases will almost certainly be needed when using paths as 
neither of the field names in the path captures the meaning of the result
and autogenerating the name would yield long names and would be dependent
on the human language in use. For that reason we may require aliases to be
specified.


### Navigating unions

Present proposal does not support navigating through unions as that would
require a way to branch paths and aggregate differing resulting data into
a single output. This would likely end up being very verbose and, thus,
be of questionable value.

There are two special cases, however.

The first case is compatibility with [RFC: In-data errors](InDataErrors.md)
That approach uses unions to embed errors inside data if they occur and would
prevent the functionality from this RFC here (safe navigation) from being
used.

To enable this, we need to support navigating through unions that that have
exactly one non-error type. The following schema lists examples:

```GraphQL
error ServerError { ... }
error RequestError { ... }

union Individual = Person

union Subject = Person | Business

extend type Query {
    # Navigabl & countable
    a: Person
    b: Individual 
    c: union Person
    d: union Individual
    e: union Person | ServerError

    # Not navigable as there is nothing to navigate
    f: Float
    g: String
    h: ID
    i: Int
    j: Boolean
    # other scalars too

    # Not navigable due to union ambiguity
    k: Subject                               # two non-error types 
    l: union Person | Business | ServerError # two non-error types
    m: union ServerError | RequestError      # no non-error types
}
```

Note, however, that all are countable (would support the suffix operator).

Additional challenge with in-data errors and shortcuts is that they can
occur "in the middle of the path" and, as such, in the type contexdt that
is different from and likely incompatible with the effective shortcut type,
if we were to do nothing about that.

Fortunately, with [RFC: Enhanced unions](EnhancedUnions.md) and 
[RFC: Anonymous types](AnonymousTypes.md) we can gather all possible
error types along the path and have the effective path result type be
an anonymous union of apparent/declared type and all *error* types along
the way,


The other special case is filtering based on type. That is not the topic
of this RFC and will only be illustrated as a possibility for a future RFC,
assuming `PaperBrochure` is a `type` implementing the `Brochure` that is
now `interface` in the following example:

```GraphQL
query {
    x1b: products?.brochure~PaperBrochure?.authors?.name?.given
    #                      ^^^^^^^^^^^^^^
}
```

### 

### Generic (framework) implementations are possible

GraphQL frameworks can implement support for all these operators generically,
without the need to change the code of the server using those frameworks.

Look-ahead optimizations are, however, welcome and could yield additional
significant performance improvements.

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
query SafeNavigationOperatorSupportQuery {
    schemaDescription: __schema?.description
}
```

Such a query would fail if introspection is disabled but, at that point, it is questionable
what such tools can do and how they would offset the lack of schema discovery. 


## Addendum 1: Solving the 64-bit long integer type

JSON treats all numbers the same and doesn't restrict the number of digits. It does 
treat them as base 10, however.

Many languages handle 64-bit integers natively and do so via JSON. 
Unfortunately, JavaScript didn't have the type that could natively store
64-bit integers safely. As it is an important player in the GraphQL ecosystem,
GraphQL limits the `Int` type to 64 bits and advises the use of the `Float`
type otherwise. `Float` type is not sufficient to store large 64-bit integers
accurately. 

After all that, JavaScript standards added support for `BigInt` but its literal
representation is incompatible with just about anything else out there, including
`JSON` as standardized - it requires `n` as a suffix. `123` is a `Number`
(no distinction between) whereas `123n` is specifically `BigInt`.

That means that, unless we force JavaScript clients to parse JSON like
everyone else does and not just evaluate it as a subset of JavaScript,
we need to use one or more different representations of 64-bit integers,
such as strings or arrays. Alternatively some may (have) introduce(d) 
custom scalars as follows:

```GraphQL

scalar JavaScriptBigInt
scalar StandardLong

type Long {
    javascript: JavaScriptBigInt
    standard: StandardLong
}

type Query {
    number: Long
}
```

The problem with the above is that it turns what is otherwise a simple scalar
into a structure that needs to be navigated. With this RFC that stops being
an issue. Non-JavaScript clients could issue the following query:

```GraphQL
query {
    number: number?.standard
}
```

whereas JavaScript clients could instead do:

```GraphQL
query {
    number: number?.javascript
}
```

While this addendum certainly isn't the core purpose of this RFC, it
may as well be treated as a(n) RFC of its own - this is a very common case.

## Addendum 2: Unit conversion example

Following the same pattern as in addendum 1, consider the following schema:

```GraphQL
type Temperature {
    K: Float
    C: Float
    F: Float
}

type Query {
    currentTemperature: Temperature
}
```
We could do:

```GraphQL
query {
    temperature: currentTemperature?.C
}
```

Unlike the following approach:

```GraphQL
enum TemperatureUnit {
    KELVIN
    CELSIUS
    FARENHEIT
}

type Query {
    currentTemperature(unit: TemperatureUnit): Float
}
```

the approach presented in this RFC has the following advantages:

- Can more easily request multiple units: `temperature: currentTemperature { C K }`

- Can retain the unit information in the value if so desired (in structure form)

- Does not require an input argument or its processing.

## Addendum 3: Time zones

Following the same pattern as in addendum 1, consider the following schema:

```GraphQL
scalar ISO8601Timestamp
scalar TimeZoneId

type Timestamp {
    utc: ISO8601Timestamp
    gmt: ISO8601Timestamp # not quite the same: see leap seconds

    # Using features from other RFCs/proposals:
    # - input unions from InputUnions.md together with 
    # - enhancements as noted in EnhancedUnions.md
    # - anonymous types from AnonymousTypes.md
    in(zone: union TimeZoneId | Int): ISO8601Timestamp    
}

type Query {
    currentTime: Timestamp
}
```

## Addendum 4: Server-side calculator

This example leverages features from other RFCs. With the following schema:

```GraphQL

union Value = Int | Long | Float | Roman

type Number {    
    plus(x: [Value!]): Number
    minus(x: [Value!]): Number
    times(x: [Value!]): Number
    dividedBy(x: [Value!]): Number
    asFloat: Float
    asRoman: Roman
}

type Query {
    sum(x: [Value!]): Number
    product(x: [Value!]): Number
}
```

The following becomes possible:

```GraphQL
query {
    result: sum(x: { Int: 5 })?.times(x: { Roman: "XIV" })?.asRoman
}
```

