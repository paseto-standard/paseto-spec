# Paseto Version 5

## GetNonce

Throw an exception. We don't do this in version 5.

## Encrypt

Given a message `m`, key `k`, and optional footer `f` (which defaults to empty
string), and an optional implicit assertion `i` (which defaults to empty string):

1. Before encrypting, first assert that the key being used is intended for use
   with `v5.local` tokens, and has a length of 256 bits (32 bytes).
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. Set header `h` to `v5.local.`
   **Note**: This includes the trailing period.
3. Generate 32 random bytes from the OS's CSPRNG to get the nonce, `n`.
4. Split the key into an Encryption key (`Ek`) and Authentication key (`Ak`),
   using HKDF-HMAC-SHA384, with `n` appended to the info rather than the salt.
    * The output length **MUST** be 48 for both key derivations.
    * The derived key will be the leftmost 32 bytes of the first HKDF derivation.

   The remaining 16 bytes of the first key derivation (from which `Ek` is derived)
   will be used as a counter nonce (`n2`):
   ```
   tmp = hkdf_sha384(
       len = 48,
       ikm = k,
       info = "paseto-encryption-key" || n,
       salt = NULL
   );
   Ek = tmp[0:32]
   n2 = tmp[32:]
   Ak = hkdf_sha384(
       len = 48,
       ikm = k,
       info = "paseto-auth-key-for-aead" || n,
       salt = NULL
   );
   ```
5. Encrypt the message using `AES-256-CTR`, using `Ek` as the key and `n2` as the nonce.
   We'll call the encrypted output of this step `c`:
   ```
   c = aes256ctr_encrypt(
       plaintext = m,
       nonce = n2
       key = Ek
   );
   ```
6. Pack `h`, `n`, `c`, `f`, and `i` together using
   [PAE](Common.md#authentication-padding)
   (pre-authentication encoding). We'll call this `preAuth`.
7. Calculate HMAC-SHA384 of the output of `preAuth`, using `Ak` as the
   authentication key. We'll call this `t`.
8. If `f` is:
    * Empty: return "`h` || base64url(`n` || `c` || `t`)"
    * Non-empty: return "`h` || base64url(`n` || `c` || `t`) || `.` || base64url(`f`)"
    * ...where || means "concatenate"
    * Note: `base64url()` means Base64url from RFC 4648 without `=` padding.

## Decrypt

Given a message `m`, key `k`, and optional footer `f`
(which defaults to empty string), and an optional
implicit assertion `i` (which defaults to empty string):

1. Before decrypting, first assert that the key being used is intended for use
   with `v5.local` tokens, and has a length of 256 bits (32 bytes).
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.

   Implementations **MUST** bind the algorithm selection to the key, not the header.
2. If `f` is not empty, implementations **MAY** verify that the value appended
   to the token matches some expected string `f`, provided they do so using a
   constant-time string compare function.
    * If `f` is allowed to be a JSON-encoded blob, implementations **SHOULD** allow
      users to provide guardrails against invalid JSON tokens.
      See [this document](../02-Implementation-Guide/01-Payload-Processing.md#optional-footer)
      for specific guidance and example code.
3. Verify that the message begins with `v5.local.`, otherwise throw an
   exception. This constant will be referred to as `h`.
    * **Note**: This includes the trailing period.
    * **Future-proofing**: If a future PASETO variant allows for encodings other
      than JSON (e.g., CBOR), future implementations **MAY** also permit those
      values at this step (e.g. `v5c.local.`).
4. Decode the payload (`m` sans `h`, `f`, and the optional trailing period
   between `m` and `f`) from base64url to raw binary. Set:
    * `n` to the leftmost 32 bytes
    * `t` to the rightmost 48 bytes
    * `c` to the middle remainder of the payload, excluding `n` and `t`
5. Split the key (`k`) into an Encryption key (`Ek`) and an Authentication key
   (`Ak`), `n` appended to the HKDF info.
    * For encryption keys, the **info** parameter for HKDF **MUST** be set to
      **paseto-encryption-key**.
    * For authentication keys, the **info** parameter for HKDF **MUST** be set to
      **paseto-auth-key-for-aead**.
    * The output length **MUST** be 48 for both key derivations.
      The leftmost 32 bytes of the first key derivation will produce `Ek`, while
      the remaining 16 bytes will be the AES nonce `n2`.

   ```
   tmp = hkdf_sha384(
       len = 48,
       ikm = k,
       info = "paseto-encryption-key" || n,
       salt = NULL
   );
   Ek = tmp[0:32]
   n2 = tmp[32:]
   Ak = hkdf_sha384(
       len = 48,
       ikm = k,
       info = "paseto-auth-key-for-aead" || n,
       salt = NULL
   );
   ```
6. Pack `h`, `n`, `c`, `f`, and `i` together (in that order) using
   [PAE](Common.md#authentication-padding).
   We'll call this `preAuth`.
7. Recalculate HMAC-SHA-384 of `preAuth` using `Ak` as the key. We'll call this `t2`.
8. Compare `t` with `t2` using a constant-time string compare function. If they
   are not identical, throw an exception.
    * You **MUST** use a constant-time string compare function to be compliant.
      If you do not have one available to you in your programming language/framework,
      you MUST use [Double HMAC](https://paragonie.com/blog/2015/11/preventing-timing-attacks-on-string-comparison-with-double-hmac-strategy).
    * Common utilities that were not intended for cryptographic comparisons, such as
      Java's `Array.equals()` or PHP's `==` operator, are explicitly forbidden.
9. Decrypt `c` using `AES-256-CTR`, using `Ek` as the key and `n2` as the nonce,
   then return the plaintext.
   ```
   return aes256ctr_decrypt(
       cipherext = c,
       nonce = n2
       key = Ek
   );
   ```

## Sign

Given a message `m`, 32-byte ML-DSA-87 seed `sk` (which will be expanded into a
full 2560-byte key at runtime), an optional footer `f` (which defaults to empty
string), and an optional implicit assertion `i` (which defaults to empty 
string):

1. Before signing, first assert that the key being used is intended for use
   with `v5.public` tokens, and is the secret key of the intended keypair.
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. Set `h` to `v5.public.`  
   **Note**: This includes the trailing period.
3. Pack `pk`, `h`, `m`, `f`, and `i` together using
   [PAE](Common.md#authentication-padding)
   (pre-authentication encoding). We'll call this `m2`.
    * Note: `pk` is the public key corresponding to `sk`. `pk` **MUST** be
      2592 bytes long.
4. Sign `m2` using ML-DSA-87 with the private key `sk`. We'll call this `sig`.  
   The output of `sig` MUST be 4627 bytes long.
   
   ```
   sig = mldsa87_sign(
       message = m2,
       secret_key_seed = sk
   );
   ```
5. If `f` is:
    * Empty: return "`h` || base64url(`m` || `sig`)"
    * Non-empty: return "`h` || base64url(`m` || `sig`) || `.` || base64url(`f`)"
    * ...where || means "concatenate"
    * Note: `base64url()` means Base64url from RFC 4648 without `=` padding.

## Verify

Given a signed message `sm`, ML-DSA-87 public key `pk` (which is 2592 bytes 
long), and optional footer `f` (which defaults to empty string), and an 
optional implicit assertion `i` (which defaults to empty string):

1. Before verifying, first assert that the key being used is intended for use
   with `v5.public` tokens, and is the public key of the intended keypair.
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
   
   Implementations **MUST** bind the algorithm selection to the key, not the header.
2. If `f` is not empty, implementations **MAY** verify that the value appended
   to the token matches some expected string `f`, provided they do so using a
   constant-time string compare function.
3. Verify that the message begins with `v5.public.`, otherwise throw an
   exception. This constant will be referred to as `h`.  
   **Note**: This includes the trailing period.
4. Decode the payload (`sm` sans `h`, `f`, and the optional trailing period
   between `m` and `f`) from base64url to raw binary. Set:
    * `s` to the rightmost 4627 bytes
    * `m` to the leftmost remainder of the payload, excluding `s`
5. Pack `pk`, `h`, `m`, `f`, and `i` together (in that order) using PAE (see
   [PAE](Common.md#authentication-padding).
   We'll call this `m2`.
    * `pk` **MUST** be 4627 bytes long.
6. Use ML-DSA-87 to verify that the signature is valid for the message:
   ```
   valid = mldsa87_verify(
       signature = s,
       message = m2,
       public_key = pk
   );
   ```
7. If the signature is valid, return `m`. Otherwise, throw an exception.
