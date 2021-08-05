# API / User Experience Recommendations

PASETO libraries should be easy to use, misuse-resistant, and secure by default.

## Builder and Parsers

The PASETO specification only covers the cryptography and message format. This section
aims to provide guidance and recommendations for how user-facing token Builders and
Parsers should be constructed.

A **Builder** is a function or class that assembles an encrypted or signed PASETO token 
at runtime.

A **Parser** is a function or class that verifies the cryptographic integrity of a
PASETO token, and also allows users to assert that the claims in the token conform to
their expectations via Parser Rules.

### Ease of Use

We have no specific recommendations on ease-of-use, since every programming language is subtly
different from each other.

### Misuse Resistance

Every Builder and Parser **SHOULD** only accept one version of the PASETO protocol and one
purpose. If you want to parse different versions and/or purposes, this **SHOULD** require
different instances at runtime. These instances **MAY** be the same underlying code (i.e.
a class) if your programming language and/or framework supports it.

Implementations **MAY** [parse the footer for Key IDs](01-Payload-Processing.md#key-id-support)
before processing the token (and, in the process, support more than one key for a given
Builder or Parser). Libraries that offer Key ID support **MUST** ensure all keys registered
in the Builder/Parser's keyring match the expected key and purpose of the Builder/Parser.
This responsibility **MUST NOT** be shifted to the user.

#### Why Constrain Builders and Parsers to One (Version, Purpose) Tuple?

Each Builder/Parser should limit the runtime negotiation as much as possible.

If you're only ever vending `v4.local` tokens, and you unexpectedly receive a `v3.local`
or `v4.public` token, whether it's a mistake or an active attack, it should be treated as
invalid input.

If applications want to support multiple versions/purposes simultaneously, they can make
the deliberate decision to do so by mapping the version and purpose from an incoming
token to the corresponding Builder/Parser instance.

The library **SHOULD NOT** provide version routing out-of-the-box. 
It's trivial to write code to map `v4.local.` token headers to a `ParserV4Local` object 
(or a `Parser` object instantiated with `ProtocolV4` and `LocalPurpose`) at runtime.

Library authors **MAY** provide version routing out-of-the-box as a separate
package/module. We recommend enforcing a strict allow-list on the versions and purposes
permitted by a version router. Implementations **MUST** fail closed (e.g. throw an Exception)
if a user provides a (version, purpose) for which the application has not specified a key.

### Secure Defaults

Libraries are encouraged to provide secure defaults for the claims in your Builders and Parsers,
through which ever mechanism is most suitable for the language and framework you're developing in.

#### Default Expiration Claims

Builders and Parsers **SHOULD**, by default, include an `exp` claim if the user did not specify
an expiration time. Users **MUST** have some means of explicitly declaring non-expiring tokens.

The recommended default expiration time is **1 hour**.

#### Why Set a Default Expiration Claim?

The use case for a non-expiring claim in the world of security tokens is fairly limited.
If someone omits an expiration header from their tokens--or forgets to require one when
processing them--this is almost certainly an oversight rather than a deliberate choice.

When it *is* a deliberate choice, users should be given the opportunity to deliberately
remove this claim from the Builder and Parser.

#### Other Default Claims

Implementations are free to choose any additional default values for [registered claims](04-Claims.md#registered-claims)
for their Parsers and Builders, but they are not required.

The default values for `iat` and `nbf` (if included in the list of default claims by the
library, and not overriden or opted out by the user) **MUST** be the current time.

PASETO libraries **MUST NOT** add default values for non-registered claims. These are
considered Custom Claims and should be left to each application to define and/or require.
