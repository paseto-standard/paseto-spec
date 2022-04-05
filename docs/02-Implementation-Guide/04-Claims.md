# Registered Claims

## Payload Claims

The following keys are reserved for use within PASETO. Users SHOULD NOT write
arbitrary/invalid data to any keys in a top-level PASETO in the list below:

| Key   | Name             | Type     | Example                                                   |
| ----- | ---------------- | -------- | --------------------------------------------------------- |
| `iss` | Issuer           | string   | `{"iss":"paragonie.com"}`                                 |
| `sub` | Subject          | string   | `{"sub":"test"}`                                          |
| `aud` | Audience         | string   | `{"aud":"pie-hosted.com"}`                                |
| `exp` | Expiration       | DateTime | `{"exp":"2039-01-01T00:00:00+00:00"}`                     |
| `nbf` | Not Before       | DateTime | `{"nbf":"2038-04-01T00:00:00+00:00"}`                     |
| `iat` | Issued At        | DateTime | `{"iat":"2038-03-17T00:00:00+00:00"}`                     |
| `jti` | Token Identifier | string   | `{"jti":"87IFSGFgPNtQNNuw0AtuLttPYFfYwOkjhqdWcLoYQHvL"}`  |

In the table above, DateTime means a string in [RFC 3339 Section 5.6 Internet Date/Time
Format](https://datatracker.ietf.org/doc/html/rfc3339#section-5.6) (a profile of ISO 8601).
Further, date and time MUST be separated with an uppercase "T", and "Z" MUST be capitalized when
used as a time offset.  While arbitrary fractional seconds SHOULD be supported in parsers, is
RECOMMENDED to omit them on creation.  Time offsets confer no special meaning, so they MUST NOT be
used for validation (beyond determining the correct moment in time).  It is RECOMMENDED that all
DateTime claims use "Z" as a time offset.

Any other claims can be freely used. These keys are only reserved in the top-level
JSON object.

The keys in the above table are case-sensitive.

Implementors SHOULD provide some means to discourage setting invalid/arbitrary data
to these reserved claims.

## Optional Footer Claims

The optional footer **MAY** contain an optional JSON object encoded as a UTF-8 string.
It does not have to be JSON. Refer to [this document](01-Payload-Processing.md#optional-footer)
for safe JSON handling recommendations, especially for preprocessing the JSON string before
parsing it into memory.

If the optional footer does contain JSON, the following claims may be stored in the footer.
Users SHOULD NOT write arbitrary/invalid data to any keys in a top-level PASETO in the list below: 

| Key   | Name           | Type   | Example                                                         |
| ----- | -------------- | ------ | --------------------------------------------------------------- |
| `kid` | Key ID         | string | `{"kid":"k4.lid.iVtYQDjr5gEijCSjJC3fQaJm7nCeQSeaty0Jixy8dbsk"}` |
| `wpk` | Wrapped PASERK | string | `{"wpk":"k4.local-wrap.pie.pu-fBxw... [truncated] ...0eo8iCS"}` |

Any other claims can be freely used. These keys are only reserved in the top-level
JSON object (if the footer contains a JSON object).

The keys in the above table are case-sensitive.

Implementors SHOULD provide some means to discourage setting invalid/arbitrary data
to these reserved claims.

#### Wrapped PASERK

Some types of serialized keys may be stored in the footer.

See [PASERK](https://github.com/paseto-standard/paserk) for more information. 
