# NIP-07: `window.nostr`

The following methods are defined for exposing NIP-44 v3 capabilities via `window.nostr`:

```
async window.nostr.nip44v3.encrypt(pubkey: string, kind: number, scope: string, plaintext: ArrayBuffer): string // encrypts a plaintext with NIP-44 v3
async window.nostr.nip44v3.decrypt(pubkey: string, kind: number, scope: string, ciphertext: string): ArrayBuffer // decrypts a ciphertext with NIP-44 v3
```
