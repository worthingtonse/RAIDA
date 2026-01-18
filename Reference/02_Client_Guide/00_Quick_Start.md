# Client Quick Start Guide

## Overview

This guide walks through implementing a minimal RAIDA client.

## Prerequisites

Before implementing:
1. Read `../../Specification/04_Cryptography/02_Key_Derivation.md` - Understand key sources
2. Read `../../Specification/04_Cryptography/00_CRC32.md` - CRC32 implementation
3. Read `../../Specification/01_Fundamentals/03_Parallel_Requests.md` - Sending to all RAIDA

## Minimal Implementation Checklist

### Step 1: Core Utilities

```python
# 1. CRC32 calculation (zlib compatible)
import zlib
import struct

def calculate_crc32(data: bytes) -> bytes:
    crc = zlib.crc32(data) & 0xFFFFFFFF
    return struct.pack('>I', crc)  # Big-endian

# 2. Challenge generation
import os

def generate_challenge() -> bytes:
    random_bytes = os.urandom(12)
    crc = calculate_crc32(random_bytes)
    return random_bytes + crc
```

### Step 2: Header Building

```python
def build_header_32(
    raida_id: int,
    command_group: int,
    command_code: int,
    encryption_type: int,
    key_dn: int,
    key_sn: int,
    body_length: int,
    echo: bytes = None
) -> bytes:
    if echo is None:
        echo = os.urandom(2)

    header = bytearray(32)
    header[0] = 0x01              # Cloud ID
    header[1] = 0x00              # Split ID
    header[2] = raida_id
    header[3] = 0x00              # Shard ID
    header[4] = command_group
    header[5] = command_code
    header[6] = 0x00              # Token type
    header[7] = encryption_type
    header[8] = key_dn
    header[9:13] = key_sn.to_bytes(4, 'big')
    header[13:16] = body_length.to_bytes(3, 'big')
    header[16:18] = echo
    # Bytes 18-31: Nonce (random for encryption)
    header[18:32] = os.urandom(14)

    return bytes(header)
```

### Step 3: Encryption

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend

def aes_ctr_encrypt(plaintext: bytes, key: bytes, nonce: bytes) -> bytes:
    # Pad nonce to 16 bytes
    counter_block = nonce.ljust(16, b'\x00')

    cipher = Cipher(
        algorithms.AES(key),
        modes.CTR(counter_block),
        backend=default_backend()
    )
    encryptor = cipher.encryptor()
    return encryptor.update(plaintext) + encryptor.finalize()

def pad_to_block(data: bytes, block_size: int = 16) -> bytes:
    remainder = len(data) % block_size
    if remainder == 0:
        return data
    padding = block_size - remainder
    return data + (b'\x00' * padding)
```

### Step 4: Echo Command (Test First)

```python
def send_echo(raida_id: int, host: str) -> bool:
    # Build request
    header = build_header_32(
        raida_id=raida_id,
        command_group=0,
        command_code=0,
        encryption_type=0,
        key_dn=0,
        key_sn=0,
        body_length=18
    )
    challenge = generate_challenge()
    body = challenge + b'\x3E\x3E'
    packet = header + body

    # Send
    response = send_udp_request(host, raida_id, packet)

    # Validate
    return response[2] == 250  # Status byte
```

### Step 5: Detect Command

```python
def send_detect(raida_id: int, host: str, token) -> bool:
    challenge = generate_challenge()

    # Token record: 1 + 4 + 16 = 21 bytes
    token_record = bytes([token.denomination])
    token_record += token.serial_number.to_bytes(4, 'big')
    token_record += token.an[raida_id]

    body = challenge + token_record
    padded_body = pad_to_block(body)

    # Encrypt
    key = token.an[raida_id]
    nonce = os.urandom(14)
    encrypted = aes_ctr_encrypt(padded_body, key, nonce)

    # Build header
    header = build_header_32(
        raida_id=raida_id,
        command_group=1,
        command_code=10,
        encryption_type=1,
        key_dn=token.denomination,
        key_sn=token.serial_number,
        body_length=len(padded_body)
    )
    # Insert nonce into header bytes 18-31
    header = header[:18] + nonce + header[32:]

    packet = header + encrypted + b'\x3E\x3E'
    response = send_udp_request(host, raida_id, packet)

    return response[2] == 241  # ALL_PASS
```

## Testing Order

1. **Echo to all 25 RAIDA** - Verify connectivity
2. **Detect single token** - Verify encryption works
3. **Detect multiple tokens** - Verify batch processing
4. **POWN single token** - Verify ownership change

## Common First-Time Errors

| Symptom | Likely Cause |
|---------|--------------|
| Status 37 | CRC32 wrong (check byte order) |
| Status 25 | Encryption key not found |
| Status 33 | Missing terminator bytes |
| Status 16 | Body length mismatch |
| Timeout | Wrong port or host |

## Next Steps

- `01_Token_Lifecycle.md` - Full token operations
- `02_Error_Recovery.md` - Handling failures
- `03_Healing_Guide.md` - Fixing fractured tokens
