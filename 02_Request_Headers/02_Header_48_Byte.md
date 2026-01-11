# 48-Byte Request Header

## Overview

The 48-byte header is used with AES-256 encryption (types 4 and 5).
It extends the 32-byte header with additional key material.

## When to Use

| Encryption Type | Header Size | Key Size |
|-----------------|-------------|----------|
| 0 (None) | 32 bytes | N/A |
| 1 (AES-128) | 32 bytes | 16 bytes |
| 4 (AES-256 single) | 48 bytes | 32 bytes |
| 5 (AES-256 dual) | 48 bytes | 32 bytes |

## Byte Map

```
Offset: 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
        CI SP RI SH CG CC TI EN DN SN SN SN SN SZ SZ SZ
        EC EC NN NN NN NN NN NN NN NN NN NN NN NN NN NN
        D2 S2 S2 S2 S2 RR RR RR RR RR RR RR RR RR RR RR
```

## Field Reference

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 1 | CI | Cloud ID (0x01) |
| 1 | 1 | SP | Split ID (0x00) |
| 2 | 1 | RI | Target RAIDA ID (0-24) |
| 3 | 1 | SH | Shard ID (0x00) |
| 4 | 1 | CG | Command Group |
| 5 | 1 | CC | Command Code |
| 6 | 1 | TI | Token ID (0x00) |
| 7 | 1 | EN | Encryption Type (4 or 5) |
| 8 | 1 | DN | Key Token 1 Denomination |
| 9-12 | 4 | SN | Key Token 1 Serial Number (BE) |
| 13-15 | 3 | SZ | Body Size (BE) |
| 16-17 | 2 | EC | Client Echo |
| 18-31 | 14 | NN | Nonce (for AES-CTR) |
| 32 | 1 | D2 | Key Token 2 Denomination (Type 5 only) |
| 33-36 | 4 | S2 | Key Token 2 Serial Number (Type 5 only) |
| 37-47 | 11 | RR | Reserved (padding) |

## Type 4: Single Token 256-bit Key

For Type 4 encryption, the key is derived from a single token:

```python
def get_type4_key(token, raida_id: int) -> bytes:
    if token.denomination in [7, 8, 9, 10, 11]:  # Key tokens
        # Key token has 32-byte secret
        return token.secret[raida_id]  # 32 bytes
    else:
        # Standard token: derive via SHA-256
        an = token.an[raida_id]  # 16 bytes
        return hashlib.sha256(an).digest()  # 32 bytes
```

## Type 5: Dual Token 256-bit Key

For Type 5 encryption, the key combines two token ANs:

```python
def get_type5_key(token1, token2, raida_id: int) -> bytes:
    an1 = token1.an[raida_id]  # 16 bytes
    an2 = token2.an[raida_id]  # 16 bytes
    return an1 + an2  # 32 bytes concatenated
```

## Building a 48-Byte Header

```python
def build_header_48(
    raida_id: int,
    command_group: int,
    command_code: int,
    encryption_type: int,  # 4 or 5
    key_token1_dn: int,
    key_token1_sn: int,
    key_token2_dn: int = 0,  # Only for type 5
    key_token2_sn: int = 0,  # Only for type 5
    body_length: int = 0,
    echo: bytes = None
) -> bytes:
    if echo is None:
        echo = os.urandom(2)

    header = bytearray(48)

    # Standard fields (same as 32-byte)
    header[0] = 0x01              # Cloud ID
    header[1] = 0x00              # Split ID
    header[2] = raida_id
    header[3] = 0x00              # Shard ID
    header[4] = command_group
    header[5] = command_code
    header[6] = 0x00              # Token type
    header[7] = encryption_type   # 4 or 5
    header[8] = key_token1_dn
    header[9:13] = key_token1_sn.to_bytes(4, 'big')
    header[13:16] = body_length.to_bytes(3, 'big')
    header[16:18] = echo
    header[18:32] = os.urandom(14)  # Nonce

    # Extended fields for 48-byte header
    if encryption_type == 5:
        header[32] = key_token2_dn
        header[33:37] = key_token2_sn.to_bytes(4, 'big')
    # Bytes 37-47 remain as zero padding

    return bytes(header)
```

## Example: Type 5 Encryption

```
Two tokens:
  Token 1: DN=1, SN=102205, AN[0]=a1a1a1...
  Token 2: DN=1, SN=102206, AN[0]=b2b2b2...

Key = a1a1a1... + b2b2b2... = 32 bytes

Header (48 bytes):
01 00 00 00 01 14 00 05  // CI SP RI SH CG=1 CC=20 TI EN=5
01 00 01 8F 3D 00 00 32  // DN1 SN1 BODYSIZE
EC EC NN NN NN NN NN NN  // Echo + Nonce
NN NN NN NN NN NN NN NN
01 00 01 8F 3E 00 00 00  // DN2 SN2 + padding
00 00 00 00 00 00 00 00
```

## Common Mistakes

**DON'T** use 48-byte header with Type 1:
```python
# WRONG - Type 1 only needs 32 bytes
header = build_header_48(encryption_type=1, ...)
```

**DO** match header size to encryption type:
```python
# CORRECT
if encryption_type in [4, 5]:
    header = build_header_48(encryption_type=encryption_type, ...)
else:
    header = build_header_32(encryption_type=encryption_type, ...)
```
