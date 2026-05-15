# NIP-44 v3

NIP-44 v3 is a new encryption scheme for Nostr that is more secure and flexible.

Compared to NIP-44 v2, it provides:
- Protection against context rebinding attacks, where the same encrypted message may be used in another context it is not intended for.
- The capability to encrypt binary data, instead of just UTF-8.
- Access control support for signers, which can restrict which kinds' data can be encrypted/decrypted, and prevent [ransom attacks](https://nostr.at/nevent1qvzqqqqqqypzqun2rcnpe3j8ge6ws2z789gm8wcnn056wu734n6fmjrgmwrp58q3qyghwumn8ghj7mn0wd68ytnvv9hxgtcpzamhxue69uhhyetvv9ujuvrcvd5xzapwvdhk6tcqyrjwy5ar5xknegcnryssk73kdt08sj7zyemam7n0r9lptemc6j3njtxpn28)
- Support for encrypted plaintexts above 65KB.

> [!NOTE]
> This specification is a **draft**, but is only expected to undergo minor, non-breaking changes.

## Get started

- Specification: [nip44v3.md](./nip44v3.md)
- Implementation guide: [implementing.md](./implementing.md).
- Test vectors: [test-vectors.json](./test-vectors.json)
- NIP-07/46 signer extensions: [extensions.md](./extensions.md)

## Implementations

| Name | Language | License |
|------|----------|---------|
| [ncrypt](https://github.com/nostr-land/ncrypt-go) | Go | BSD-3 |