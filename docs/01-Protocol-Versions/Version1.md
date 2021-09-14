# Paseto Version 1

## GetNonce

Given a message (`m`) and a nonce (`n`):

1. Calculate HMAC-SHA384 of the message `m` with `n` as the key.
2. Return the leftmost 32 bytes of step 1.

## Encrypt

Given a message `m`, key `k`, and optional footer `f`
(which defaults to empty string):

1. Before encrypting, first assert that the key being used is intended for use
   with `v1.local` tokens, and has a length of 256 bits (32 bytes).
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. Set header `h` to `v1.local.`
3. Generate 32 random bytes from the OS's CSPRNG.
4. Calculate `GetNonce()` of `m` and the output of step 2
   to get the nonce, `n`.
   * This step is to ensure that an RNG failure does not result
     in a nonce-misuse condition that breaks the security of
     our stream cipher.
5. Split the key into an Encryption key (`Ek`) and Authentication key (`Ak`),
   using the leftmost 16 bytes of `n` as the HKDF salt:
   ```
   Ek = hkdf_sha384(
       len = 32
       ikm = k,
       info = "paseto-encryption-key",
       salt = n[0:16]
   );
   Ak = hkdf_sha384(
       len = 32
       ikm = k,
       info = "paseto-auth-key-for-aead",
       salt = n[0:16]
   );
   ```
6. Encrypt the message using `AES-256-CTR`, using `Ek` as the key and
   the rightmost 16 bytes of `n` as the nonce. We'll call this `c`:
   ```
   c = aes256ctr_encrypt(
       plaintext = m,
       nonce = n[16:]
       key = Ek
   );
   ```
7. Pack `h`, `n`, `c`, and `f` together using
   [PAE](Common.md#authentication-padding)
   (pre-authentication encoding). We'll call this `preAuth`
8. Calculate HMAC-SHA384 of the output of `preAuth`, using `Ak` as the
   authentication key. We'll call this `t`.
9. If `f` is:
   * Empty: return "`h` || base64url(`n` || `c` || `t`)"
   * Non-empty: return "`h` || base64url(`n` || `c` || `t`) || `.` || base64url(`f`)"
   * ...where || means "concatenate"
   * Note: `base64url()` means Base64url from RFC 4648 without `=` padding.

## Decrypt

Given a message `m`, key `k`, and optional footer `f`
(which defaults to empty string):

1. Before decrypting, first assert that the key being used is intended for use
   with `v1.local` tokens. See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. If `f` is not empty, implementations **MAY** verify that the value appended
   to the token matches some expected string `f`, provided they do so using a
   constant-time string compare function.
3. Verify that the message begins with `v1.local.`, otherwise throw an
   exception. This constant will be referred to as `h`.
4. Decode the payload (`m` sans `h`, `f`, and the optional trailing period
   between `m` and `f`) from base64url to raw binary. Set:
   * `n` to the leftmost 32 bytes
   * `t` to the rightmost 48 bytes
   * `c` to the middle remainder of the payload, excluding `n` and `t`
5. Split the key (`k`) into an Encryption key (`Ek`) and an Authentication key
   (`Ak`), using the leftmost 16 bytes of `n` as the HKDF salt.
   * For encryption keys, the **info** parameter for HKDF **MUST** be set to
     **paseto-encryption-key**.
   * For authentication keys, the **info** parameter for HKDF **MUST** be set to
     **paseto-auth-key-for-aead**.
   * The output length **MUST** be 32 for both keys.
   
   ```
   Ek = hkdf_sha384(
       len = 32
       ikm = k,
       info = "paseto-encryption-key",
       salt = n[0:16]
   );
   Ak = hkdf_sha384(
       len = 32
       ikm = k,
       info = "paseto-auth-key-for-aead",
       salt = n[0:16]
   );
   ```
6. Pack `h`, `n`, `c`, and `f` together (in that order) using
   [PAE](Common.md#authentication-padding).
   We'll call this `preAuth`.
7. Recalculate HMAC-SHA-384 of `preAuth` using `Ak` as the key. We'll call this
   `t2`.
8. Compare `t` with `t2` using a constant-time string compare function. If they
   are not identical, throw an exception.
9. Decrypt `c` using `AES-256-CTR`, using `Ek` as the key and the rightmost 16
   bytes of `n` as the nonce, and return this value.
   ```
   return aes256ctr_decrypt(
       cipherext = c,
       nonce = n[16:]
       key = Ek
   );
   ```

## Sign

Given a message `m`, 2048-bit RSA secret key `sk`, and
optional footer `f` (which defaults to empty string):

1. Before signing, first assert that the key being used is intended for use
   with `v1.public` tokens, and is the secret key of the intended keypair.
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. Set `h` to `v1.public.`
3. Pack `h`, `m`, and `f` together using
   [PAE](Common.md#authentication-padding)
   (pre-authentication encoding). We'll call this `m2`.
4. Sign `m2` using RSA with the private key `sk`. We'll call this `sig`.
   ```
   sig = crypto_sign_rsa(
       message = m2,
       private_key = sk,
       padding_mode = "pss",
       public_exponent = 65537,
       hash = "sha384"
       mgf = "mgf1+sha384"
   );
   ```
   Only the above parameters are supported. PKCS1v1.5 is explicitly forbidden.
5. If `f` is:
   * Empty: return "`h` || base64url(`m` || `sig`)"
   * Non-empty: return "`h` || base64url(`m` || `sig`) || `.` || base64url(`f`)"
   * ...where || means "concatenate"
   * Note: `base64url()` means Base64url from RFC 4648 without `=` padding.

## Verify

Given a signed message `sm`, RSA public key `pk`, and optional
footer `f` (which defaults to empty string):

1. Before verifying, first assert that the key being used is intended for use
   with `v1.public` tokens, and is the public key of the intended keypair.
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. If `f` is not empty, implementations **MAY** verify that the value appended
   to the token matches some expected string `f`, provided they do so using a
   constant-time string compare function.
3. Verify that the message begins with `v1.public.`, otherwise throw an
   exception. This constant will be referred to as `h`.
4. Decode the payload (`sm` sans `h`, `f`, and the optional trailing period
   between `m` and `f`) from base64url to raw binary. Set:
   * `s` to the rightmost 256 bytes
   * `m` to the leftmost remainder of the payload, excluding `s`
5. Pack `h`, `m`, and `f` together (in that order) using PAE (see
   [PAE](Common.md#authentication-padding).
   We'll call this `m2`.
6. Use RSA to verify that the signature is valid for the message:
   ```
   valid = crypto_sign_rsa_verify(
       signature = s,
       message = m2,
       public_key = pk,
       padding_mode = "pss",
       public_exponent = 65537,
       hash = "sha384"
       mgf = "mgf1+sha384"
   );
   ```
7. If the signature is valid, return `m`. Otherwise, throw an exception.
