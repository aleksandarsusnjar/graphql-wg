# RFC: Introspectable directives

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar)

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on** [RFC: Allow directives on directives](DirectivesOnDirectives.md)

There have been multiple attempts to expose additional/custom schema metadata inside introspection in a variety of ways. Meanwhile, many ended up communicating the schema directly, bypassing introspection, to gain access to directives.

> This is an alternate (counter) proposal to the [RFC: Annotation Structs](AnnotationStructs.md) and, previously, [RFC: Metadata Structs](MetadataStructs.md).

The proposed approach can make the status quo valid in a clean way, by defining a new directive:

```
"""
Exposes the annotated element (directive) in introspection.
"""
directive @graphql.introspectable on DIRECTIVE_DEFINITION
``` 

Additional introspection schema is required:

```GraphQL
extend type __Schema {
   schemaDirectiveDefinitions: [graphql.SchemaDirectiveDefinition!]!
}

extend type __Type {
  schemaDirectives: [graphql.SchemaDirective!]!
}

extend type __Field {
  schemaDirectives: [graphql.SchemaDirective!]!
}

extend type __EnumValue {
  schemaDirectives: [graphql.SchemaDirective!]!
}

extend type __Directive {
  schemaDirectives: [graphql.SchemaDirective!]!
}

namespace graphql {
  type SchemaDirectiveDefinition {
    name: String!
    id: ID!
    namespace: Namespace
    description: String
    locations: [__DirectiveLocation!]!
    args: [__InputValue!]!
    isRepeatable: Boolean!
  }

  type SchemaDirective {    
    definition: SchemaDirectiveDefinition
    description: String
    locations: [__DirectiveLocation!]!
    args: [__InputValue!]!
    isRepeatable: Boolean!
  }
}
```

