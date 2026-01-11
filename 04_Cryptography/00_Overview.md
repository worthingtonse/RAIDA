# Cryptography Overview

## Encryption Types

| Code | Name | Key Size | Header Size | Key Source |
|------|------|----------|-------------|------------|
| 0 | None | N/A | 32 bytes | N/A |
| 1 | AES-128 Shared | 128-bit | 32 bytes | Token AN |
| 2 | AES-128 Locker | 128-bit | 32 bytes | Locker code |
| 3 | AES-128 RKE | 128-bit | 32 bytes | RAIDA key ID |
| 4 | AES-256 Shared | 256-bit | 48 bytes | 256-bit secret |
| 5 | AES-256 Dual | 256-bit | 48 bytes | Two token ANs |
| 7 | AES-256 RAIDA | 256-bit | 48 bytes | RAIDA secret |

## Cipher

All encryption uses **AES in CTR (Counter) mode**.

- Block size: 16 bytes (for both AES-128 and AES-256)
- Nonce: Taken from header nonce field
- No authentication tag (challenge-response provides integrity)

## What Gets Encrypted

| Component | Encrypted? |
|-----------|------------|
| Request Header | No |
| Request Body | Yes (if EN > 0) |
| Terminator (0x3E3E) | No |
| Response Header | No |
| Response Body | No |

## Key Derivation

See `02_Key_Derivation.md` for detailed key derivation rules.

CRITICAL: Different encryption types derive keys differently:
- Type 1: 16-byte token AN directly
- Type 4: TRUE 256-bit secret (NOT duplicated AN)
- Type 5: Concatenation of two 16-byte ANs

## Decision Tree

```
Need encryption?
  |
  NO --> Type 0
  |
  YES --> Have token?
            |
            NO --> Type 2 (locker) or Type 3 (RKE)
            |
            YES --> Need 256-bit?
                      |
                      NO --> Type 1
                      |
                      YES --> Have two tokens?
                                |
                                NO --> Type 4 (256-bit secret)
                                |
                                YES --> Type 5 (dual key)
```
