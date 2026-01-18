# Cryptography Overview

## Encryption Types

| Code | Name | Key Size | Header Size | Key Source | Status |
|------|------|----------|-------------|------------|--------|
| 0 | None | N/A | 32 bytes | N/A | Supported |
| 1 | AES-128 Token | 128-bit | 32 bytes | Token AN | Supported |
| 2 | AES-128 Locker | 128-bit | 32 bytes | Locker AN | Supported |
| 3 | Reserved | N/A | 32 bytes | N/A | Unsupported |

**Note:** Type 3 is reserved for future use. Clients MUST NOT use it. Servers will reject with `ERROR_INVALID_ENCRYPTION`.

## Cipher

All encryption uses **AES-128 in CTR (Counter) mode**.

- Key size: 128 bits (16 bytes)
- Block size: 16 bytes
- Nonce: 8 bytes from header, padded to 16 bytes with zeros
- No authentication tag (challenge-response provides integrity)

## What Gets Encrypted

| Component | Encrypted? |
|-----------|------------|
| Request Header | No |
| Request Body (excluding terminator) | Yes (if EN > 0) |
| Terminator (0x3E3E) | No |
| Response Header | No |
| Response Body | No |

## Key Source

| Type | Key Source | Description |
|------|------------|-------------|
| 0 | None | No encryption |
| 1 | Token AN | 16-byte Authenticity Number of the specified token |
| 2 | Locker AN | 16-byte AN from locker index lookup |

## Nonce Construction

For Types 1 and 2, the server constructs a 16-byte nonce for AES-CTR:

```
Header bytes 24-31:  [N0][N1][N2][N3][N4][N5][N6][N7]
                                              ^Echo^
Internal 16-byte:    [N0][N1][N2][N3][N4][N5][N6][N7][00][00][00][00][00][00][00][00]
```

- Bytes 0-7: Copied from header bytes 24-31
- Bytes 8-15: Zeros

**Important:** Header bytes 30-31 serve dual purpose as both nonce bytes 6-7 AND echo bytes.

## Decision Tree

```
Need encryption?
  |
  NO --> Type 0 (unencrypted, but challenge still required)
  |
  YES --> Using locker?
            |
            YES --> Type 2 (locker AN)
            |
            NO --> Type 1 (token AN)
```

## Challenge Requirement

All types (0, 1, 2) require a 16-byte challenge at the start of the request body:

- **Type 0:** Challenge sent in plaintext, CRC validated directly
- **Types 1/2:** Challenge encrypted, after decryption: `challenge_hash = decrypted[0:16] XOR key`

See `00_CRC32.md` and `01_Challenge_Construction.md` in this folder for challenge construction details.
