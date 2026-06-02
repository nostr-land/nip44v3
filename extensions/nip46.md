## NIP-46: Nostr Remote Signing

## Methods

Remote signers or clients that want to use NIP-44 v3 should use the following methods:

### `nip44v3_encrypt`

This method accepts 4 arguments:
- `<pubkey>`: The other public key, as hex.
- `<kind>`: The kind to use.
- `<scope>`: The scope to use.
- `<plaintext_base64>`: The plaintext to encrypt, base64-encoded.

This method returns one value:
- `<ciphertext>`: The ciphertext as a string.

### `nip44v3_decrypt`

This method accepts 4 arguments:
- `<pubkey>`: The other public key, as hex.
- `<kind>`: The kind that is expected.
- `<scope>`: The scope that is expected.
- `<ciphertext>`: The ciphertext to decrypt.

This method returns one value:
- `<plaintext_base64>`: The plaintext, as a base64 encoded string.

## Notes

Implementations should ensure that the plaintext is base64 encoded.
