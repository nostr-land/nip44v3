# Extensions to signer specifications

NIP-44 v3 requires special handling for the `kind` and `scope` fields, which requires changes to NIPs relating to signers.

## NIP-07

The following functions should be used to expose NIP-44 v3 functionality.

```
async window.nostr.nip44v3.encrypt(pubkey: string, kind: number, scope: string, plaintext: ArrayBuffer): string // encrypts a plaintext with NIP-44 v3
async window.nostr.nip44v3.decrypt(pubkey: string, kind: number, scope: string, ciphertext: string): ArrayBuffer // decrypts a ciphertext with NIP-44 v3
```

## NIP-46

The following methods are defined for NIP-44 v3 encryption support for remote signers:

### `nip44v3_encrypt`

Takes in 4 arguments:
- `<pubkey>`: The other public key, as hex.
- `<kind>`: The kind to use.
- `<scope>`: The scope to use.
- `<plaintext_base64>`: The plaintext to encrypt, base64-encoded.

Returns one value:
- `<ciphertext>`: The ciphertext as a string.

### `nip44v3_decrypt`

Takes in 4 arguments:
- `<pubkey>`: The other public key, as hex.
- `<kind>`: The kind that is expected.
- `<scope>`: The scope that is expected.
- `<ciphertext>`: The ciphertext to decrypt.

Returns one value:
- `<plaintext_base64>`: The plaintext, as a base64 encoded string.
