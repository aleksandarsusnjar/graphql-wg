# RFC: Privacy and security directives

**Proposed by**: [Aleksandar Susnjar](https://github.com/aleksandarsusnjar), others welcome to join.

**Part of:** [RFC: Comprehensive Enhancements](ComprehensiveEnhacement.md)

**Requires / builds on:**
- [RFC: Namespace support](Namespacing.md)


```GraphQL
"""
Specifies that the data is sensitive and should not be stored beyond immediate use, cached or logged.
"""
directive @graphql.secret on SCALAR, OBJECT, FIELD_DEFINITION, INPUT_FIELD_DEFINITION, INTERFACE, UNION, ENUM, INPUT_OBJECT

directive @graphql.sensitive
```

## `graphql.lettercase`

This is relevant to external handling of equivalence of values and is especially needed for aggregation tools/proxies having to match or distinguish values and compute set unions.

```GraphQL
"""
Specifies the text handling semantics for a String or ID field or a scalar definition.
"""
directive @graphql.text.meta(
    """
    Indicates whether lettercase differences affect value equality.
    """
    caseSensitive: Boolean!

    unicodeNormalization: org.unicode.NormalizationForm

    """
    Indicates whether beginning and ending whitespace affects value equality.
    """
    paddingSensitive: Boolean!

    """
    Indicates whether internal whitespace affects value equality.
    """
    internalWhitespace: Boolean!

    """
    Optionally specifies the regular expression validation - all must match.
    """
    validation: [graphql.text.RegularExpressionCondition!]
) on SCALAR, INPUT_FIELD_DEFINITION, ARGUMENT_DEFINITION, FIELD_DEFINITION

namespace org.unicode {
    enum NormalizationForm {
        """
        Canonical Decomposition
        """
        NFD

        """
        Canonical Decomposition, followed by Canonical Composition
        """
        NFC

        """
        Compatibility Decomposition
        """
        NFKD

        """
        Compatibility Decomposition, followed by Canonical Composition
        """
        NFKC
    } @specifiedBy(url: "https://www.unicode.org/reports/tr15")
} 
``` 