# NIP-17: Private Direct Messages

## Receivers

A client which wishes to signal for NIP-44 v3 support for receiving should add a single `encryption` tag to the *`10050` Private Message Relays* event.

This tag contains a space separated list of supported encryption standards, which can be:
- `nip44_v3`: NIP-44 v3
- `nip44_v2`: *Legacy* NIP-44 v2

Clients should list all supported methods. When decrypting, the payload can be decoded to determine the version.

Example:
```jsonc
{
    "kind": 10050,
    "tags": [
        ["encryption", "nip44_v3 nip44_v2"],
        ["relay", "wss://nostr.land"]
    ],
    "content": "",
    // ...
}
```

## Senders

Senders should look at the `encryption` tag and use the best standard available when sending.

If there is no encryption tag, it should be assumed to be `nip44_v2` only.

## Encryption

The following kind and scope should be used for encryption and decryption:
- Gift Wraps (`1059`): Kind `1059`, and an empty scope (`""`).
- Seals (`13`): Kind `13`, and an empty scope (`""`).