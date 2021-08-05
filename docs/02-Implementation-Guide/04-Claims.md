# Registered Claims

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

In the table above, DateTime means an ISO 8601 compliant DateTime string.

Any other claims can be freely used. These keys are only reserved in the top-level
JSON object.

The keys in the above table are case-sensitive.

Implementors SHOULD provide some means to discourage setting invalid/arbitrary data
to these reserved claims.
