# RFC: Input expressions that enable powerful DSLs

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Namespace support](Namespacing.md)
- [RFC: Operation Types](OperationTypes.md)
- [RFC: Allow directives on directives](DirectivesOnDirectives.md)
- [RFC: Introspectable directives](IntrospectableDirectives.md)
- [RFC: Input Unions](InputUnions.md)

**See also:**
- [RFC: Type references](TypeReferences.md)


## Problem Statement

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

## Compatibility Considerations

Legacy clients never encountered and, thus, never requested this feature.
Additionally, they would be able to use the legacy (deep-structured) syntax 
to pass input expressions.

New clients will either be hard-coded to use the new feature or will notice the
`@operator` directives as introspectable and enable the use of expressions.