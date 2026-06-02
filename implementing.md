# Implementing NIP-44 v3

## For protocol designers

### Context

When setting the context for encryption, you should almost always use the event kind of the parent event. A "fake kind" should be **reserved** if used, and should still be within the allowed range of `0-65535`.

The scope is an *optional* UTF-8 string that can be used to add further scoping to the encrypted data. It should be derived from the *most obvious* item in the event.

A good example is for people-lists, to set the `scope` to the `d` tag of the list.
A bad example is setting the `scope` to the `npub` for a DM protocol. This is redundant as the encrypted message is already bound to the recipient, and it adds no useful data.

Do *not* try to parse the context from the message yourself, and instead determine the context from the event itself and pass it to the decryption function. Unless you work on tools to inspect encrypted data, this usually means that you are misusing NIP-44 v3.

The context should not be used to store additional data not in the outer event. It instead serves as a protection against attacks, and a mechanism for access control.

### Binary data and message sizes

It is recommended to use binary data when possible for items such as encrypted lists of public keys. This should be done in a way that can be extended in the future.

Compression should be avoided unless there is a very good reason, as [CRIME](https://en.wikipedia.org/wiki/CRIME)-style attacks may cause leakage of the encrypted data.

Do not use NIP-44 v3 to encrypt files, as it is not designed for random access or streaming, and requires loading the entire message into RAM.

## For signer developers

When developing a signer, ensure that you can support kinds above `65535` when encrypting and decrypting. You should show these kinds to the user as is.

Use access control on `scope` only for kinds that your signer knows. Do not process `scope`s for unknown kinds, and instead ask for permission for the whole `kind`.

Where relevant, offer a "only allow kind + scope" option.

See [extensions](./extensions) on the commands / interfaces for NIP-44 v3 support that you can add to your signer.

## For library developers

The rest of this document will focus on library implementation.

The following are best practices:
- Always accept a context, verify it yourself, and do not expect the user to verify one that you return.
    - It is possible to accidentally ignore the context, or verify it improperly.
    - Use cases that require decrypting with an unknown context should use a method to extract the context data from a message. These methods should be marked to be possibly dangerous or unsafe.
- Ensure that your library supports kinds above `65535`.
- Do not canonicalize or otherwise transform the provided `scope`, and reject `scope`s that are not valid UTF-8.
- Make sure that your library can encrypt/decrypt binary data, not just UTF-8 strings.
- Do not allow users to specify a custom nonce. This is required for the test vectors, but should be a strictly internal API that is disabled in a build.

## Simplified explanation

All encodings use big endian.

Encrypted messages are Base64 encoded versions of the following binary format:
```
[u8 - version (0x03)]
[32 bytes - nonce]
[32 bytes - mac]
[u32 - kind]
[u32 - scope length]
[variable length scope]
[rest of data: ciphertext]
```

After following the key derivation process, the MAC can be calculated as the HMAC of the following data using the `mac_key`:
```
[32 bytes - nonce]
[u32 - kind]
[u32 - scope]
[variable length scope]
[variable length ciphertext] 
```

The ciphertext is encrypted with ChaCha20 using the derived `encryption_key`, and an all-zeroes nonce. The decrypted data looks as follows:

```
[u32 - plaintext length]
[variable length plaintext]
[0x00 padding]
```

The padding should be checked to be all-zeroes. Implementations **must not** do any other checks on the padding length.

There is no constraint on how the padded length is decided, but implementations **should** use the algorithm in `spec.md` to prevent fingerprinting.

There is no limit on the plaintext length. However, it is recommended to avoid long plaintexts, as they are harder to transmit through relays, and must be decrypted fully in RAM.

## Test vectors

The test vectors of NIP-44 v3 are available in [test-vectors.json](./test-vectors.json). These are composed of 4 different tests:

### Encrypt/Decrypt

This tests the key derivation, encryption, and decryption routines of your app.

Each entry in the `encrypt_decrypt` array contains the following fields:
- `secret1` / `secret2`: The private key of both parties, as hex.
- `nonce`: The 32-byte nonce, as hex.
- `kind`: The kind to use in the context.
- `scope_hex`: The scope to use in the context, as hex.
- `plaintext_hex`: The plaintext, as hex.
- `ciphertext`: The ciphertext.
- `encryption_key`: The derived encryption key as hex.
- `mac_key`: The derived MAC key as hex.
- `prk`: The HKDF PRK as hex.

From both the perspective of secret1 and secret2, test the following:
- Check that your parser have parsed the correct `nonce`, `kind` and `scope`
- Check that key derivation produces the correct `encryption_key`, `mac_key` and `prk`
- Decrypt the `ciphertext` and ensure it matches `plaintext_hex`
- Encrypt the `plaintext_hex` with the given `nonce` and context, and ensure it matches `ciphertext`.

### Decrypt Only

This section includes some ciphertexts that are intentionally non-standard, but are otherwise allowed by the specification. Implementations should be able to successfully decrypt these.

Each item in `decrypt_only` has a similar format to Encrpyt/Decrypt, with the following fields:
- `secret1` / `secret2`: The private key of both parties, as hex.
- `nonce`: The 32-byte nonce, as hex.
- `kind`: The kind to use in the context.
- `scope_hex`: The scope to use in the context, as hex.
- `plaintext_hex`: The plaintext, as hex.
- `ciphertext`: The ciphertext.
- `encryption_key`: The derived encryption key as hex.
- `mac_key`: The derived MAC key as hex.
- `prk`: The HKDF PRK as hex.
- `note`: A note for this specific test case.

From both the perspective of secret1 and secret2, test the following:
- Check that your parser have parsed the correct `nonce`, `kind` and `scope`
- Check that key derivation produces the correct `encryption_key`, `mac_key` and `prk`
- Decrypt the `ciphertext` and ensure it matches `plaintext_hex`

### Long Encrypt/Decrypt

This tests the encryption behavior under long tests.

Each entry in the `long_encrypt_decrypt` array contains the following fields:
- `secret1` / `secret2`: The private key of both parties, as hex.
- `nonce`: The 32-byte nonce, as hex.
- `kind`: The kind to use in the context.
- `scope_hex`: The scope to use in the context, as hex.
- `pattern_hex`: The pattern bytes, as hex.
- `repeat`: The number of times to repeat the pattern.
- `ciphertext_sha256`: The ciphertext's SHA256 hash.

The plaintext to use is the `pattern` bytes repeated `repeat` times.

From both the perspective of secret1 and secret2, test the following:
- Encrypt the calculated plaintext with the given `nonce` and context, and ensure its hash is equal to `ciphertext_sha256`
- Decrypt the `ciphertext` you encrypted, and ensure it matches the computed plaintext

### Invalid Decryption

The invalid decryption test contains intentionally malformed inputs.

Each entry in the `invalid_decryption` array contains the following fields:
- `secret`: The private key to decrypt as.
- `public`: The public key of the other party.
- `kind`: The kind to use in the context.
- `scope_hex`: The scope to use in the context, as hex.
- `ciphertext`: The ciphertext.
- `why`: why this ciphertext is invalid.

You should check the errors from your own application to ensure that it is rejecting for the same reason as `why`.

Test that the decryption of `ciphertext` fails.
