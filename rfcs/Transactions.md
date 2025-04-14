#  RFC: Transactions

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar)

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Namespace support](Namespacing.md)
- [RFC: Allow directives on directives](DirectivesOnDirectives.md)
- [RFC: Introspectable directives](IntrospectableDirectives.md)
- [RFC: Operation Types](OperationTypes.md)

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

## Introspection 

Transactable operations could be listed within `graphql.Prepare` type, equivalently
to how fields are listed in `graphql.Operation`, `Query`, `Mutation` and `Subscription`
types.

## Separate data area


## Compatibility Considerations

### Legacy client + new server

Legacy clients did not have the ability to execute transactions.
Their requests will continue to be the same.

### New client + legacy server

New clients would fail to prepare the transactions using the `prepare` operations
and can use that as a proble to check for support.

