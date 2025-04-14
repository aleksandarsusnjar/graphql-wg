#  RFC: Operation Types

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar)

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Namespace support](Namespacing.md)

**Relied upon by:**
 - [RFC: Transactions](Transactions.md)


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

> **Additional note:** servers that can determine inter-dependency of individual steps (or lack thereof) can
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

A syntactic element, a symbol or a keyword is needed to mark this. Here `*` is proposed as it is easy to type on most keyboards and is often used as a wildcard implying flexibility. Multiple proposed alternatives follow.

#### Alternative 1: Unordered fragment

A special inline fragment is introduced that is by updating [2.8.2 Inline Fragments](https://spec.graphql.org/draft/#sec-Inline-Fragments) from:

> *InlineFragment*:
>
> ... TypeCondition? Directives? SelectionSet

to:

> *InlineFragment*:
>
> ... (TypeCondition | '%') ? Directives? SelectionSet

... such that when `%` is used instead of the type condition it retains the type of the outer context but states that the order of fields does not need to be maintained.

```GraphQL
operation {
  subject(id: 123) {      
    __typename
    id
    ...% { # Server may return inner fields in any order
        name
        location
    }
  }
}
```

> **Note:** may also be applied to ordered-execution blocks `[`...`]` as execution may need to be in order
> but the response serialization does not.

#### Alternative 2: Ordered/unordered divider

The `%%%` is chosen to mean "output field order is not important after this point" and is placed between the fields and/or fragments in the requests. For example:

```GraphQL
operation {
  subject(id: 123) {      
    __typename
    id
    %%% # Server may return remaining fields in any order
    name
    location
  }
}
```

> **Note:** may also be applied to ordered-execution blocks `[`...`]` as execution may need to be in order
> but the response serialization does not.

Do note that keywords cannot be used here as they may end up conflicting with field names and directives would be attached to a field, not the space between, and may move together with the field during merging.

#### Alternative 3: Unordered block/fragment

Consider the simplification of the previous alternative where `%%%` is only permitted to be at the very beginning of a block and nowhere else. The same outcome can be accomplished using anonymous (or other) fragments:

```GraphQL
operation {
  subject(id: 123) {
    __typename
    id
    ... {% # Server may return remaining fields in any order
      name
      location
    %}
  }
}
```

The example above ends the block in a similar fashion, though this may be optional. Angle brackets `<`...`>` or  
a brace decorator character other than `%` could be used. However, we may want to avoid ending the block in 
`?}`, `!}`, `+}` to avoid clashing with [RFC: Safe navigation (shortcut) and refactoring operators](SafeNavigationAndRefactoring.md).

> **Note:** may also be applied to ordered-execution blocks `[`...`]` as execution may need to be in order
> but the response serialization does not.


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
operation OperationSupportDiscovery {
    worksIfSucceeds: __typeName
}
```

Such a query would fail if introspection is disabled but, at that point, it is questionable
what such tools can do and how they would offset the lack of schema discovery. 