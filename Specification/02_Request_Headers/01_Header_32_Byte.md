# 32-Byte Request Header

## Usage

Used with encryption types 0, 1, and 2.

**Note:** Type 3 is reserved for future use and is not currently supported. Servers will reject requests with encryption type 3.

## Byte Map

```
Offset: 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
        VR SP RI SH CG CM ID ID BF AP AP CP TR AI PI PC
        EN DN SN SN SN SN BL BL NO NO NO NO NO NO EC EC
```

## Bytes 16-31 At A Glance

The meaning of bytes 16-31 varies by encryption type:

| Offset | Size | Type 0 (None) | Type 1 (AES) | Type 2 (Locker) |
|--------|------|---------------|--------------|-----------------|
| 16 | 1 | EN = 0x00 | EN = 0x01 | EN = 0x02 |
| 17 | 1 | Ignored | Key DN | Key DN |
| 18-21 | 4 | Ignored | Key SN | Key SN |
| 22-23 | 2 | Body Length | Body Length | Body Length |
| 24-29 | 6 | Ignored | Nonce[0-5] | Nonce[0-5] |
| 30-31 | 2 | Echo | Nonce[6-7] + Echo | Nonce[6-7] + Echo |

**Key:** DN = Denomination, SN = Serial Number (big-endian)

## Field Reference (Bytes 0-15)

| Offset | Size | Field | Description | Value |
|--------|------|-------|-------------|-------|
| 0 | 1 | VR | Version | 0x01 |
| 1 | 1 | SP | Split ID | 0x00 (reserved) |
| 2 | 1 | RI | RAIDA ID | 0-24 |
| 3 | 1 | SH | Shard ID | 0x00 (reserved) |
| 4 | 1 | CG | Command Group | See commands |
| 5 | 1 | CM | Command Code | See commands |
| 6-7 | 2 | ID | Coin ID | 0x0001 (big-endian) |
| 8 | 1 | BF | Bitfield | 0x01 |
| 9-10 | 2 | AP | Application Port | 0x0000 (reserved) |
| 11 | 1 | CP | Compression | 0x00 (reserved) |
| 12 | 1 | TR | Translation | 0x00 (reserved) |
| 13 | 1 | AI | AI Mode | 0x00 (reserved) |
| 14 | 1 | PI | Packet Index | 0x00 |
| 15 | 1 | PC | Packet Count | 0x01 |

---

## Type 0: Unencrypted

For unencrypted requests, the key fields are ignored but the challenge is still required in the body.

### Header Fields (Bytes 16-31)

| Offset | Size | Field | Description | Value |
|--------|------|-------|-------------|-------|
| 16 | 1 | EN | Encryption Type | 0x00 |
| 17-21 | 5 | -- | Ignored | 0x00 |
| 22-23 | 2 | BL | Body Length | Size in bytes (big-endian) |
| 24-29 | 6 | -- | Ignored | 0x00 |
| 30-31 | 2 | EC | Echo | Client tracking value |

### Body Format

```
[Challenge 16B][Command Payload][Terminator 0x3E3E]
```

The challenge is sent in plaintext. Server validates CRC directly.

### Python Example

```python
import os
import zlib
import struct

def build_type0_header(raida_id: int, command_group: int, command_code: int,
                       body_length: int, echo: bytes) -> bytes:
    header = bytearray(32)

    # Routing (0-7)
    header[0] = 0x01              # Version
    header[1] = 0x00              # Split ID
    header[2] = raida_id
    header[3] = 0x00              # Shard ID
    header[4] = command_group
    header[5] = command_code
    header[6:8] = (1).to_bytes(2, 'big')  # Coin ID

    # Presentation (8-15)
    header[8] = 0x01              # Bitfield
    header[9:16] = b'\x00' * 7    # Reserved

    # Encryption section (16-31) - Type 0
    header[16] = 0x00             # Encryption type
    header[17:22] = b'\x00' * 5   # Ignored
    header[22:24] = body_length.to_bytes(2, 'big')
    header[24:30] = b'\x00' * 6   # Ignored
    header[30:32] = echo[:2]      # Echo bytes

    return bytes(header)

def build_type0_body(command_payload: bytes) -> bytes:
    # Generate challenge: 12 random + 4-byte CRC32 (big-endian)
    random_data = os.urandom(12)
    crc = zlib.crc32(random_data) & 0xFFFFFFFF
    challenge = random_data + struct.pack('>I', crc)

    return challenge + command_payload + b'\x3e\x3e'
```

---

## Type 1: AES-128 (Token Key)

For encrypted requests using a token's AN as the AES-128 key.

### Header Fields (Bytes 16-31)

| Offset | Size | Field | Description | Value |
|--------|------|-------|-------------|-------|
| 16 | 1 | EN | Encryption Type | 0x01 |
| 17 | 1 | DN | Key Denomination | Token denomination |
| 18-21 | 4 | SN | Key Serial Number | Token SN (big-endian) |
| 22-23 | 2 | BL | Body Length | Encrypted size (big-endian) |
| 24-31 | 8 | NO | Nonce | 8 random bytes |
| 30-31 | 2 | EC | Echo (dual-purpose) | Client tracking value |

**Important:** Bytes 30-31 serve dual purpose:
- Last 2 bytes of the 8-byte nonce (used for AES-CTR decryption)
- Echo bytes (returned in server response)

Client guidance: Use random bytes for 24-29, and your tracking value for 30-31.

### Body Format

```
[Encrypted: Challenge 16B + Command Payload][Terminator 0x3E3E]
```

**Note:** The terminator (0x3E3E) is NOT encrypted.

### Python Example

```python
import os
import zlib
import struct
from Crypto.Cipher import AES
from Crypto.Util import Counter

def build_type1_header(raida_id: int, command_group: int, command_code: int,
                       key_denomination: int, key_serial: int,
                       body_length: int, nonce_8bytes: bytes) -> bytes:
    header = bytearray(32)

    # Routing (0-7)
    header[0] = 0x01
    header[1] = 0x00
    header[2] = raida_id
    header[3] = 0x00
    header[4] = command_group
    header[5] = command_code
    header[6:8] = (1).to_bytes(2, 'big')

    # Presentation (8-15)
    header[8] = 0x01
    header[9:16] = b'\x00' * 7

    # Encryption section (16-31) - Type 1
    header[16] = 0x01                              # Encryption type
    header[17] = key_denomination & 0xFF           # Denomination (int8)
    header[18:22] = key_serial.to_bytes(4, 'big')  # Serial number
    header[22:24] = body_length.to_bytes(2, 'big') # Body length

    # Nonce (24-31) - 8 bytes total
    # Bytes 30-31 also serve as echo
    header[24:32] = nonce_8bytes[:8]

    return bytes(header)

def build_type1_body(command_payload: bytes, key_an: bytes,
                     nonce_8bytes: bytes) -> bytes:
    # Server uses 16-byte nonce: 8 from header + 8 zeros
    nonce_16 = nonce_8bytes[:8] + b'\x00' * 8

    # Generate challenge: 12 random + 4-byte CRC32
    random_data = os.urandom(12)
    crc = zlib.crc32(random_data) & 0xFFFFFFFF
    challenge = random_data + struct.pack('>I', crc)

    # Plaintext = challenge + payload
    plaintext = challenge + command_payload

    # Encrypt with AES-128-CTR
    ctr = Counter.new(128, initial_value=int.from_bytes(nonce_16, 'big'))
    cipher = AES.new(key_an[:16], AES.MODE_CTR, counter=ctr)
    ciphertext = cipher.encrypt(plaintext)

    # Terminator is NOT encrypted
    return ciphertext + b'\x3e\x3e'
```

---

## Type 2: AES-128 (Locker Key)

Type 2 has an identical header structure to Type 1. The only difference is how the server looks up the encryption key:

- **Type 1:** Key looked up in regular token page store
- **Type 2:** Key looked up in locker index

### Header Fields (Bytes 16-31)

| Offset | Size | Field | Description | Value |
|--------|------|-------|-------------|-------|
| 16 | 1 | EN | Encryption Type | 0x02 |
| 17 | 1 | DN | Locker Denomination | Locker denomination |
| 18-21 | 4 | SN | Locker Serial Number | Locker SN (big-endian) |
| 22-23 | 2 | BL | Body Length | Encrypted size (big-endian) |
| 24-31 | 8 | NO | Nonce | 8 random bytes |
| 30-31 | 2 | EC | Echo (dual-purpose) | Client tracking value |

### Body Format

Same as Type 1.

### Python Example

Same as Type 1, but set `header[16] = 0x02`.

---

## Type 3: Reserved

Type 3 is reserved for future use and is not currently implemented.

- Clients MUST NOT send requests with encryption type 3
- Servers WILL reject with `ERROR_INVALID_ENCRYPTION` (code 34)

---

## Challenge Requirements

All encryption types (0, 1, 2) require a 16-byte challenge at the start of the body:

| Type | Challenge Location | Validation |
|------|-------------------|------------|
| 0 | Plaintext in body | CRC checked directly |
| 1 | Encrypted in body | Decrypted, then XOR with key, CRC checked |
| 2 | Encrypted in body | Decrypted, then XOR with key, CRC checked |

The challenge format is always: `[12 random bytes][4-byte CRC32 big-endian]`

See `../04_Cryptography/01_Challenge_Construction.md` for details.

---

## Nonce Details

For Types 1 and 2, the server constructs a 16-byte internal nonce:
- Bytes 0-7: Copied from header bytes 24-31
- Bytes 8-15: Set to zeros

This 16-byte nonce is used for AES-CTR mode encryption/decryption.
