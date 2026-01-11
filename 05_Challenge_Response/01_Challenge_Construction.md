# Challenge Construction

## Purpose

The challenge field provides:
1. Mutual authentication (server proves it can decrypt)
2. Replay attack prevention (unique per request)
3. Response integrity verification

## Challenge Format

The 16-byte challenge field consists of:

```
[12 bytes random data][4 bytes CRC32]
```

| Offset | Size | Content |
|--------|------|---------|
| 0-11 | 12 bytes | Cryptographically random data |
| 12-15 | 4 bytes | CRC32 of bytes 0-11 (big-endian) |

## Construction Algorithm

```python
import os
import zlib
import struct

def generate_challenge() -> bytes:
    """
    Generate a 16-byte challenge field.

    Returns:
        16 bytes: [12 random][4 CRC32]
    """
    # Step 1: Generate 12 random bytes
    random_data = os.urandom(12)

    # Step 2: Calculate CRC32 of random data
    crc = zlib.crc32(random_data) & 0xFFFFFFFF

    # Step 3: Convert CRC to big-endian bytes
    crc_bytes = struct.pack('>I', crc)

    # Step 4: Concatenate
    return random_data + crc_bytes
```

## Placement in Request Body

The challenge is always the first 16 bytes of the request body:

```
Request Body:
[Challenge 16B][Command-specific data...][Padding][Terminator]
 ^-- bytes 0-15
```

## Example

```
Random bytes: A1 B2 C3 D4 E5 F6 07 08 09 0A 0B 0C
CRC32 of above: 0x12345678
CRC bytes (BE): 12 34 56 78

Complete challenge:
A1 B2 C3 D4 E5 F6 07 08 09 0A 0B 0C 12 34 56 78
```

## Requirements

1. **MUST** use cryptographically secure random generator
2. **MUST** generate new random bytes for each request
3. **MUST** use big-endian byte order for CRC32
4. **MUST** store challenge for response validation

## Common Mistake

**DON'T** reuse challenge bytes:
```python
# WRONG - Enables replay attacks
challenge = generate_challenge()
for raida in range(25):
    send_request(raida, challenge)  # Same challenge!
```

**DO** generate unique challenge per RAIDA:
```python
# CORRECT
challenges = {}
for raida in range(25):
    challenges[raida] = generate_challenge()
    send_request(raida, challenges[raida])
```
