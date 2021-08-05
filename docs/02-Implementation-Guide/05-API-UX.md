# API / User Experience Recommendations

PASETO libraries should be easy to use, misuse-resistant, and secure by default.

## Builder and Parsers

### Secure Defaults

Libraries are encouraged to provide secure defaults for the claims in your Builders and Parsers,
through which ever mechanism is most suitable for the language and framework you're developing in.

Builders and Parsers **SHOULD**, by default, include an `exp` claim if the user did not specify
an expiration time. Users **MUST** have some means of explicitly declaring non-expiring tokens.

The recommended default expiration time is **1 hour**.
