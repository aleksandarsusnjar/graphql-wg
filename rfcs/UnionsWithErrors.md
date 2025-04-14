# RFC: Anonymous unions and in-data errors

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Namespace support](Namespacing.md)


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
2. Allow anonymous unions of mixed type kinds, i.e. `squareRoot(arg: Float!): Float! | ComplexNumber! | MathError!`. Those can be exposed as namespaced types or can be erased and every field could be treated as returning an automatically generated and named (namespaced) union including the field name or the schema could be enhanced to support multiple output types for any one field, with the first one listed being used as a compatibility field. Note:
    - Making a union of a scalar and a complex type is considered OK as the complex (object) value would be enclosed in the `{`...`}`.
    - A union of different kinds of scalars whose values cannot be used to distinguish the type is prohibited. For example, it is OK to make a union of `Boolean`, `Int` and `String` but not any combination of `String`, `ID` and `enum`, or `Float` and `Int`, for example. Custom scalars can be combined (or not) based on their literal JSON representations.
3. Have dedicated error `interface`, `type` and `union` definitions, marked using the preceding keyword `error`, with following requirements/restrictions:
    - Error unions can only bring together error interfaces, types and error unions.
    - Non-error unions do not have this restriction.
    - Error types can have fields of non-error types.
    - Fields of non-error types can mix error and non-error types but have to have at least one non-error type listed. In union lists the non-error types must be listed first (this is not hugely important from a functionality standpoint but may help with readability).
    - The fact that a type is an error type is introspectable via the new field to be added to the `__Type` type.
4. Error values do not need to be listed in the field selection. All fields are assumed requested, including `__typename`. Fields of those errors that are of error types are also included. Fields of non-error types are not included by default and are subject to standard fragments. 
5. The requestor has to opt-in to accepting errors to be communicated this way by using `?` suffix or `?.` or `!.` joiners or explicit fragments to indicate the support for this level of GraphQL specification. Without this, the error is communicated traditionally, outside `data` but with all the error fields included within the error structure.

## Compatibility Considerations

- `__typename` will continue to indicate the concrete type as in the past.
- Legacy introspection clients will either see generated union types and deal with them as they would have normally or focus on the first type only.
- New introspection clients accessing old servers would act the same way with the generated union types or could probe for multi-type output support.