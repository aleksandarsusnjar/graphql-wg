# RFC: Regular expressions support


**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar)

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Namespace support](Namespacing.md)


Very commonly used, including here. Can help a variety of tools, including schema validators.

```GraphQL
namespace graphql.text {
  """
  Regular expression 
  """
  scalar RegularExpression

  input RegularExpressionCondition {
    """
    The pattern to match.
    """
    pattern: RegularExpression!     

    """
    Indicates whether the valid values must NOT match the specified `pattern`.
    """
    negative: Boolean! = false

    """
    Specifies the identifier of this condition, allowing tools to better indicate
    the validation failures. Note that those are ids that are meant to be definitive
    and possibly used to look up localizations.
    """
    conditionId: ID!

    """
    Optional default description.
    """
    description: String
  }
}
```

## Compatibility Considerations

This appears as a library of types. Legacy clients that never saw it, never needed it.
New clients would use it if explicitly coded for it or if they discover it in the schema.