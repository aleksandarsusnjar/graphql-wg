# RFC: Input Unions

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Anonymous types](AnonymousTypes.md)
- [RFC: Type references](TypeReferences.md)
- [RFC: Enhanced unions](EnhancedUnions.md)


**See also:**
- [RFC: GraphQL Input Union](InputUnion.md) (an older proposal)
- `@oneOf`


## Problem statement

While `@oneOf` serves as a great workaround for the issue it remains verbose and,
potentially, somewhat repetitve (type names must be repeated as input field names
in some form).

[RFC: GraphQL Input Union](InputUnion.md) presented an attempt to address some of
these issues. Proposal presented in *this* document you are currently reading 
takes advantage of other enhancements proposed in 
[RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md) to make things 
easier, cleaner and consistent.

## Proposed solution

Take [RFC: Anonymous types](AnonymousTypes.md) as a base and simply allow its
forward-looking functionality with respect to input unions. Also allow input
interfaces and inheritance with named types, as aluded possible in there, as
the solution below allows for this to be supported.

Furthermore, take [RFC: Type references](TypeReferences.md) as an inspiration
and allow type references to be used as input field identifiers.

Given that, whenever an (input) argument is of polymorphic type, i.e. either an
`input interface` or a named or anonymous `input union`, the caller/request 
must specify the concrete type in one of the alternative ways listed below, 
all sharing this example schema:

```GraphQL
union Number = Int | Float

scalar RomanNumeralsNumber

input ComplexNumber {
    real: union Number! | RomanNumeralsNumber!
    imaginary: union Number! | RomanNumeralsNumber!
}

extend type Query {
    foo(x: union Number | RomanNumeralsNumber | ComplexNumber): String
}
```

### Alternative 1: No envelope with syntactic sugar

```GraphQL
query {
    a: foo(x: Int = 123)
    b: foo(x: Float = 123.0)
    c: foo(x: RomanNumeralsNumber = "CXXIII")
    d: foo(x: ComplexNumber = { 
        real: RomanNumeralsNumber = "CXXIII"
        imaginary: Int = 0
    })
}
```

This alternative is concise and matches the Schema Definition Language syntax
but may appear unclear how it would be mapped to JSON, when query variables
(parameters) are used. In this case they would have to take form of a JSON
object literal, e.g.: for `d`:

```JSON
    "x_for_d": {
        "type": "ComplexNumber",
        "value": {
            "real": {
                "type": "RomanNumeralsNumber",
                "value": "CXXIII"
            },
            "imaginary": {
                "type": "Int",
                "value": 0
            }
        }
    }
```


### Alternative 2: Within envelope

This alternative departs from the Schema Definition Language syntax but
is more consistent with the JSON representation as in alternative 1.

```GraphQL
query {
    a: foo(x: { Int: 123 })
    b: foo(x: { Float = 123.0 })
    c: foo(x: { RomanNumeralsNumber: "CXXIII" })
    d: foo(x: {
        ComplexNumber: { 
            real: { RomanNumeralsNumber: "CXXIII" }
            imaginary: { Int: 0 }
        }
    })
}
```

The same JSON representation remains as in the alternative 1.

The benefit of this approach is that it does not require syntactic sugar
and can be accomplished by the server rewriting polymorphic input types
as anonymous (concrete) input types, i.e. the original example schema


```GraphQL
union Number = Int | Float

scalar RomanNumeralsNumber

input ComplexNumber {
    real: union Number! | RomanNumeralsNumber!
    imaginary: union Number! | RomanNumeralsNumber!
}

extend type Query {
    foo(x: union Number | RomanNumeralsNumber | ComplexNumber): String
}
```

would be automatically rewritten during loading/parsing as:

```GraphQL
union Number = Int | Float

scalar RomanNumeralsNumber

input ComplexNumber {
    real: input @oneOf {
        Int: Int!
        Float: Float!
        RomanNumeralsNumber: RomanNumeralsNumber!
    }
    imaginary: input @oneOf {
        Int: Int!
        Float: Float!
        RomanNumeralsNumber: RomanNumeralsNumber!
    }
}

extend type Query {
    foo(
        x: input @oneOf {
            Int: Int!
            Float: Float!
            RomanNumeralsNumber: RomanNumeralsNumber!
            ComplexNumber: ComplexNumber!
        }
    ): String
}
```

