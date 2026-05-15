NIP-44 v3
=========

**This specification is a draft.**

NIP-44 v3 introduces a new asymmetric encryption scheme designed for Nostr.

## Overview

This scheme provides a basic encryption primitive that can be used for encrypting and decrypting data between two parties. This spec does not define any specific usages or kinds.

NIP-44 v3 provides the following features:
- Encrypting binary plaintexts into Base64-encoded ciphertexts safe for insertion into Nostr events, and decrypting them back into the original data.
- Additional authenticated data is attached to the ciphertext in the form of the event `kind` and a custom `scope`.
    - This allows protecting against cross-context replay, and allows signers to restrict encryption/decryption access.
- Limited message size obscuring using padding.

This scheme does **NOT** provide the following:
- No forward secrecy: If one of the parties' keys are leaked, it is possible to decrypt all ciphertexts.
- No metadata hiding: The outer event, including the author and timestamps are not protected.
- No deniability: This scheme does not provide plausible deniability of encrypted messages by itself.
- Message size privacy: The **rough** size of the message is visible to anyone observing the ciphertext.

It is up to the application to provide these guarantees with its own protocols.

## Definitions

Notes:
- The type `[n]x` is an array that contains `n` elements of type `x`. The type `[]x` is an array that contains any number of elements of type `x`.
- It is considered an error to input a wrong-sized slice or the wrong type into a function.

Definitions:
- Functions
    - `split(separator: byte, bytes: []byte): [][]byte`: Splits the given byte array on the `separator` byte, and returns an array of byte arrays representing each segment.
    - `join(separator: byte, inputs: [][]byte): []byte`: Concatenates the given byte arrays, separating them with `separator`, and returns the resulting byte array.
    - `repeat(b: byte, n: int): [n]byte`: Returns a byte array with `n` bytes that are set to `b`.
    - `length(array: []any): int`: Returns the length of the given array.
    - `base64_encode(data: []byte): []byte`: Encodes the given byte array using the Base64 encoding as specified in [RFC4648].
    - `base64_decode(data: []byte): []byte`: Decodes the given byte array using the standard Base64 encoding as specified in [RFC4648]. Fails on malformed input.
    - `u32_to_bytes(n: int): [4]byte`: Converts the given unsigned 32-bit integer into 4 bytes with big endian encoding.
    - `bytes_to_u32(b: [4]byte): int`: Converts the given 4 bytes into an unsigned 32-bit integer with big endian encoding.
    - The functions `floor(value: number): int`, `ceil(value: number): int`, `min`, `max` represent well known mathematical operations.
    - `log_base(value: number, base: number): number`: Returns the base-`base` logarithm of `value`.
- Operators
    - The operators `+`, `-`, `*` and `/` represent well known mathematical operations.
    - The operator `a ** b` returns the integer `a` raised to the `b`'th power.
    - `arr_a || arr_b` returns the concatenation of the same-typed arrays `arr_a` and `arr_b` into a new array.
    - `array[i:j]` returns a new array, with all elements from `arr` starting from `i` (inclusive) and ending at `j` (exclusive).
        - Negative indices represent offsets from the end of the array and must be converted to a positive index like so: `idx + length(array)`. If the result is still negative, an out-of-bounds error is raised.
- Cryptographic functions
    - `random(n: int): [n]byte`: Returns a byte array containing `n` uniform and random bytes from a cryptographically-secure random number generator.
    - `constant_time_eq(a: [n]byte, b: [n]byte): bool`: Compares `a` and `b` for equality in constant time, and returns the result.
    - `HKDF(salt: []byte, IKM: []byte, info: []byte, L: int): [L]byte`: Computes the [HKDF] operation `HKDF-Expand(HKDF-Extract(salt, IKM), info, L)`, with the `SHA-256` hash function.
    - `HMAC(key: []byte, data: []byte): [32]byte`: Computes the [HMAC]-SHA256 function.
    - `SHA256(data: []byte): [32]byte`: Computes the SHA256 hash of the data.
    - `ChaCha20(key: [32]byte, data: [n]byte): [n]byte`: Runs [ChaCha20] using the given key and data, with a 96-bit nonce of all zeroes, and a counter starting from zero. 
    - `ECDH(seckey_a: [32]byte, pubkey_b: [32]byte): [32]byte`: Computes the scalar multiplication of the scalar `seckey_a`, and x-only point `pubkey_b` by using the following formula, using definitions from  [BIP-340]:  
    `bytes(x(int(seckey_a) ⋅ lift_x(int(pubkey_b))))`  
    It fails if the `pubkey_b` does not exist on the curve, or if `seckey_a` is not in the range `[1, n)`


### Context

Two pieces of additional, unencrypted but authenticated data can be attached to a ciphertext, known as the "context":
- `kind`, which is an unsigned 32-bit integer. This SHOULD represent the event kind the encrypted message is contained in.
- `scope`, which is an additional UTF-8 encoded string meant for additional scoping.

`kind`s above 65535 are reserved by this specification for future use, and SHOULD NOT be used by protocol developers. Libraries and implementors of NIP-44 v3 however MUST support `kind`s above 65535.

Signers SHOULD NOT try to interpret `scope` for `kind`s they do not support.

## Encryption

The encryption function accepts the following inputs:
- `seckey_a`: The private key of the encrypting party (A).
- `pubkey_b`: The public key of the **other** party (B). This may however be the same keypair as A.
- `kind`: The kind, as a 32-bit unsigned integer.
- `scope`: The scope, as a UTF-8 encoded byte array.
- `plaintext`: The plaintext to encrypt, as a byte array. Up to `2^31-1` bytes.

Encryption is performed with the following steps:
1. Generate a nonce
    - `nonce = random(32)`
2. Derive `encryption_key` and `mac_key`
    - See [Key Derivation](#key-derivation)
3. Create the prefixed plaintext.
    - `prefixed_plaintext = u32_to_bytes(length(plaintext)) || plaintext`
4. Pad the message to the target size, determined with the algorithm in section [Padding](#padding)
    - `padded_plaintext = prefixed_plaintext || repeat('\x00', target_size(length(prefixed_plaintext)) - length(prefixed_plaintext))`
5. Encrypt the message using ChaCha20
    - `chacha20_ciphertext = ChaCha20(key = encryption_key, data = padded_plaintext)`
6. Encode the authenticated data
    - `authenticated_data = nonce || u32_to_bytes(kind) || u32_to_bytes(length(scope)) || scope || chacha20_ciphertext`
7. Compute the MAC for the authenticated data
    - `mac = HMAC(key = mac_key, data = authenticated_data)`
8. Encode the final ciphertext
    - `ciphertext = base64_encode(0x03 || nonce || mac || u32_to_bytes(kind) || u32_to_bytes(length(scope)) || scope || chacha20_ciphertext)`

## Decryption

The decryption function accepts the following inputs:
- `seckey_a`: The private key of the decrypting party (A).
- `pubkey_b`: The public key of the **other** party (B). This may however be the same keypair as A.
- `expected_kind`: The kind that is expected, as a 32-bit unsigned integer.
- `expected_scope`: The scope that is expected, as a UTF-8 string.
- `ciphertext`: The ciphertext to decrypt, as a byte array.

Decryption is performed with the following steps:
1. Decode the ciphertext into its binary form
    - Fail if `length(ciphertext) == 0`
    - Fail if `ciphertext[0] == '#'` with an unsupported version error
        - This exists to allow non-Base64 encodings in the future.
    - `decoded_ciphertext = base64_decode(ciphertext)`
2. Ensure the message is well formed.
    - Fail if `length(decoded_ciphertext) < 77`
    - Fail if `decoded_ciphertext[0] != 3` with an unsupported version error
3. Reconstruct the original components
    - `nonce = decoded_ciphertext[1:33]`
    - `mac = decoded_ciphertext[33:65]`
    - `kind = bytes_to_u32(decoded_ciphertext[65:69])`
    - `scope_length = bytes_to_u32(decoded_ciphertext[69:73])`
    - Fail if `scope_length > length(decoded_ciphertext) - 73`
    - `scope = decoded_ciphertext[73:73+scope_length]`
    - `chacha20_ciphertext = decoded_ciphertext[73+scope_length:length(decoded_ciphertext)]`
    - Fail if `length(chacha20_ciphertext) < 4`
4. Check the scope and kind to be what is expected
    - Fail if `kind != expected_kind`
    - Fail if `scope != expected_scope`
5. Derive `encryption_key` and `mac_key`
    - See [Key Derivation](#key-derivation)
6. Encode the authenticated data
    - `authenticated_data = nonce || u32_to_bytes(kind) || u32_to_bytes(length(scope)) || scope || chacha20_ciphertext`
7. Check the MAC of the message
    - Fail if `constant_time_eq(mac, HMAC(key = mac_key, data = authenticated_data)) == false`
8. Decrypt the ciphertext
    - `padded_plaintext = ChaCha20(key = encryption_key, data = chacha20_ciphertext)`
9. Find the plaintext length from the message
    - `plaintext_length = bytes_to_u32(padded_plaintext[0:4])`
    - Fail if `plaintext_length + 4 > length(padded_plaintext)`
    - Fail if `plaintext_length > 2^31 - 1`
10. Validate padding is all zeroes
    - Fail if `constant_time_eq(padded_plaintext[4+plaintext_length:length(padded_plaintext)], repeat('\x00', length(padded_plaintext)-4-plaintext_length)) == false`
11. Obtain the plaintext
    - `plaintext = padded_plaintext[4:4+plaintext_length]`

## Key Derivation

The following steps should be taken to derive `encryption_key` and `mac_key` from a `seckey_a`, `pubkey_b` and `nonce` when encrypting/decrypting messages:
1. Derive the shared secret
    - `shared_secret = ECDH(seckey_a, pubkey_b)`
2. Derive the `encryption_key` and `mac_key` using HKDF
    - `encryption_key = HKDF(salt = 'nip44-v3\x00' || nonce, IKM = shared_secret, info = 'encryption_key', L = 32)`
    - `mac_key = HKDF(salt = 'nip44-v3\x00' || nonce, IKM = shared_secret, info = 'mac_key', L = 32)`

## Padding

Implementations SHOULD use the following algorithm, which is almost identical to the one in NIP-44 v2.

The constants for this algorithm are:
- `minimum_size`: 32
- `chunk_subdivs_small`: 4
- `chunk_subdivs_large`: 8
- `chunk_large_threshold`: 32768

The function `target_size(len)` that returns the target size for a message of non-zero length `len` is defined as:
1. Find the next power of 2 that is **equal or greater** than `len`.
    - `next_power = 2 ** ceil(log2(len))`
2. Compute the `chunk_subdivs`
    - If `next_power >= chunk_large_threshold`, assume `chunk_subdivs = chunk_subdivs_large`.
    - Otherwise, assume `chunk_subdivs = chunk_subdivs_small`
3. Compute the chunk size to use.
    - `chunk_size = max(minimum_size, next_power / chunk_subdivs)`
3. Pick the equal or next largest message size that is a multiple of `chunk_size`:
    - `target_size(len) = chunk_size * ceil(len / chunk_size)`

For `len = 0`, the padded length is `minimum_size`.

## Backwards/forwards compatibility

The first byte of the decoded ciphertext is the version number. This is `3` for NIP-44 v3, and was `2` for NIP-44 v2. Depending on just this byte, the correct algorithm can be selected.

To allow for extending with non-Base64 encodings, all ciphertexts starting with `#` should be treated as an *unknown future version* when decrypting.

[HKDF]: https://datatracker.ietf.org/doc/html/rfc5869
[HMAC]: https://datatracker.ietf.org/doc/html/rfc2104
[ChaCha20]: https://datatracker.ietf.org/doc/html/rfc8439
[BIP-340]: https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
[RFC4648]: https://datatracker.ietf.org/doc/html/rfc4648