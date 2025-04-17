# RFC: Echoes - reflecting outputs back to inputs

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Operation Types](OperationTypes.md)
- [RFC: Safe navigation (shortcut) and refactoring operators](SafeNavigationAndRefactoring.md)

**See also:**
- [RFC: Input Unions](InputUnions.md)
- [RFC: Type references](TypeReferences.md)
- [RFC: Enhanced unions](EnhancedUnions.md)


## Problem Statement

This RFC aims to accomplish one thing that opens up a beneficial side effect.

The "problem" in focus here is the inability to use server's responses (field outputs)
as inputs for subsequent operations without first communicating them all the way back
to the client and issuing a new request. This yields:

- Reduced performance due to requesting additional roundtrips and could be seen 
  as a form of underfetching.

- To offset that servers may opt to implement additional operations that represent
  frequent use cases, thus requiring additional server development time and 
  inconsistent implementations.

Somewhat common case could be illustrated with the following schema:

```GrahpQL
scalar BookId
scalar ISBN
scalar UPC
scalar LibraryCatalogueNumber

type Book {
    id: BookId!
    isbn: ISBN!
    upc: UPC!
    cn: LibraryCatalogueNumber!

    ...

    reviews: [Review!]!
}

type Review {
    id: BookId!
    book: Book!
    text: String
}

type Query {
    bookById(id: BookId!): Book
    bookByISBN(isbn: ISBN!): Book
    bookByUPC(upc: UPC!): Book
    bookByLibraryCatalogueNumber(cn: LibraryCatalogueNumber!): Book    
}

type Mutation {
    addReview(bookId: ID!, text: String): Review
}
```

To add a review for a book by, say its ISBN, as `addReview` requires a book `id`
instead, one would have to issue two separate requests. The first to look up
the `id` based on the known `isbn`:

```GraphQL
{
   bookByISBN(isbn: ...) { id }
}
```

and then, separately, to add the review:

```GraphQL
mutation {
   addReview(id: <value from the response>, text: "Why two requests?") { id }
}
```

There are multiple challenges here that prevent this use case with present GraphQL
specification:

1. Query fields can be handled in any order and would, in this case, need to be 
   strictly followed up a mutation. This problem is addressed in the
   [RFC: Operation Types](OperationTypes.md).

2. Only scalars and enum types can be used both as input and output types. 
   This problem is lessened by [RFC: Safe navigation (shortcut) and refactoring operators](SafeNavigationAndRefactoring.md)
   as it has the power to extract scalars from structures and sets the 
   stage for additional (future) RFCs that may provide additional refactoring
   abilities (not a goal here). Do note that GraphQL does not have
   type reflection abilities on the input side like it has for the output.
   Servers are "content" with what amounts to 
   [Duck typing](https://en.wikipedia.org/wiki/Duck_typing).

3. While there is a way to refence query parameters (called "variables", though
   there is nothing variable about them and can be used for other operation types)
   via `$varname`, there are no means to address the output.


This RFC targets the challenge #3 above, assuming #1 and #2 are and/or will be
solved by other referenced RFCs and future RFCs for further enhancements.

Do note that that, while [RFC: Operation Types](OperationTypes.md) together with
[RFC: Input Unions](InputUnions.md) and [RFC: Enhanced unions](EnhancedUnions.md)
can solve the example shown as follows:

```GrahpQL
scalar BookId
scalar ISBN
scalar UPC
scalar LibraryCatalogueNumber

type Book {
    id: BookId!
    isbn: ISBN!
    upc: UPC!
    cn: LibraryCatalogueNumber!

    ...

    reviews: [Review!]!

    addReview(text: String): Review
}

type Review {
    id: BookId!
    book: Book!
    text: String
}

type Operation {
    book(id: union BookId | ISBN! | UPC! | LibraryCatalogueNumber!): Book
}
```

with the request as follows:

```GraphQL
operation [
   book(id: { ISBN: ... }) { 
      addReview(text: "This works") {
        id
      }
   }
]
```

it continues to require servers to explicitly implement such operations.
The goal of this particular RFC is to enable clients to accomplish similar
results with existing servers updated only to use new GraphQL server 
framework versions that implement this RFC and no need to significant 
code changes, if any at all.

## Proposal

Introduce a special "query variable" `$$` that, at any point, contains the
data the current output context has produced so far. This addressing would
only be permitted in ordered execution, i.e. within the `[`...`]` operation 
blocks - see [RFC: Operation Types](OperationTypes.md) if this is unfamiliar.

Together with [RFC: Safe navigation (shortcut) and refactoring operators](SafeNavigationAndRefactoring.md)
(including `__back`), it becomes possible to issue a request as follows:

```GraphQL
[  # begins an ordered execution block as per the Operation Types RFC
   book(isbn: ...) { id }
   addReview(id: $$!.book!.id, text: "Single request works!") { id }
   #               ^^    ^^ safe navigation operators as per the referenced RFC
]
```
... or:

```GraphQL
[  # begins an ordered execution block as per the Operation Types RFC
   bid: book(isbn: ...)!.id
   #                   ^^ safe navigation operator as per the referenced RFC
   addReview(id: $$!.bid, text: "Single request works!") { id }
   #               ^^ safe navigation operators as per the referenced RFC
]
```
alternatively:
```GraphQL
[  # begins an ordered execution block as per the Operation Types RFC
   bid: book(isbn: ...)!.id
   #                   ^^ safe navigation operator as per the referenced RFC
   addReview(id: $$bid, text: "Single request works!") { id }
]
```

## Side benefits

This RFC also opens up a possibility to create GraphQL testing proxies/wrappers
for automated testing fully "coded" in GraphQL. Taking the same example schema
from above:

```GraphQL
operation TestBookLookupByISBN [
    book(isbn: 12345678) {
        id
        isbn
        upc
        cn
    }

    # following fields could be added by the proxy/wrapper and need not
    # be present in the original schema

    assertEquals(actual: $$!.book!.isbn, expected: { ISBN: 12345678 })
    assertEquals(actual: $$!.book!.id,   expected: { BookId: 345345 })
    assertEquals(actual: $$!.book!.upc,  expected: { UPC: 982374902 })
    ...
]

