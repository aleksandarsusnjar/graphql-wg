# RFC: Allow directives on directives

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Relied upon by:**
- [RFC: Introspectable Directives](IntrospectableDirectives.md)
- [RFC: Input expressions that enable powerful DSLs](InputExpressions.md)

Adding directives to directives will allow various schema meta-processing tools to introduce their own additional special processing even for directives they do not define.

Change:

```
DirectiveDefinition : Description? directive @ Name ArgumentsDefinition? `repeatable`? on DirectiveLocations
...
TypeSystemDirectiveLocation : one of
  - `SCHEMA`
  - `SCALAR`
  - `OBJECT`
  - `FIELD_DEFINITION`
  - `ARGUMENT_DEFINITION`
  - `INTERFACE`
  - `UNION`
  - `ENUM`
  - `ENUM_VALUE`
  - `INPUT_OBJECT`
  - `INPUT_FIELD_DEFINITION`
```

to (append `Directives` to DirectiveDefinition and append `DIRECTIVE_DEFINITION` to locations):

```
DirectiveDefinition : Description? directive @ Name ArgumentsDefinition? `repeatable`? on DirectiveLocations Directives
...
TypeSystemDirectiveLocation : one of
  - `SCHEMA`
  - `SCALAR`
  - `OBJECT`
  - `FIELD_DEFINITION`
  - `ARGUMENT_DEFINITION`
  - `INTERFACE`
  - `UNION`
  - `ENUM`
  - `ENUM_VALUE`
  - `INPUT_OBJECT`
  - `INPUT_FIELD_DEFINITION`
  - `DIRECTIVE_DEFINITION`
```

## Compatibility Considerations

### Legacy client + new server

Legacy clients do not see or request schema directives at all.

### New client + legacy server

This change alone changes nothing visible to the clients.
However, see [RFC: Introspectable Directives](IntrospectableDirectives.md).