# Challenge Construction

## Purpose

The challenge field provides:
1. Mutual authentication (server proves it can decrypt)
2. Replay attack prevention (unique per request)
3. Response integrity verification

## Applicability

**The challenge is REQUIRED for ALL encryption types (0, 1, and 2).**

Even unencrypted requests (Type 0) must include the challenge in the body.

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

## Validation Algorithm

```python
import zlib
import struct

def validate_challenge(challenge: bytes) -> bool:
    """
    Validate that a challenge has correct CRC.

    Args:
        challenge: 16-byte challenge to validate

    Returns:
        True if CRC matches, False otherwise
    """
    if len(challenge) != 16:
        return False

    random_data = challenge[:12]
    provided_crc = challenge[12:16]

    # Calculate expected CRC
    crc = zlib.crc32(random_data) & 0xFFFFFFFF
    expected_crc = struct.pack('>I', crc)

    return provided_crc == expected_crc
```

## Placement in Request Body

The challenge is always the first 16 bytes of the request body:

```
Request Body:
[Challenge 16B][Command-specific data...][Terminator 0x3E3E]
 ^-- bytes 0-15
```

## Handling by Encryption Type

| Type | Challenge State | Server Validation |
|------|-----------------|-------------------|
| 0 | Plaintext | CRC checked directly on body[0:16] |
| 1 | Encrypted | Decrypt body, XOR with key, check CRC |
| 2 | Encrypted | Decrypt body, XOR with key, check CRC |

### Type 0 (Unencrypted)

The challenge is sent in plaintext. The server:
1. Copies body[0:16] as the challenge
2. Computes CRC32 of body[0:12]
3. Validates it matches body[12:16]

```python
# Server validation for Type 0
challenge = body[0:16]
expected_crc = crc32(body[0:12])
actual_crc = body[12:16]
if expected_crc != actual_crc:
    return ERROR_INVALID_CRC
```

### Types 1 and 2 (Encrypted)

The challenge is encrypted along with the payload. The server:
1. Decrypts the body using AES-CTR with the token/locker AN
2. XORs decrypted[0:16] with the key to produce challenge_hash
3. Validates CRC32 of decrypted[0:12] matches decrypted[12:16]

```python
# Server validation for Types 1/2
decrypted = aes_ctr_decrypt(body, key, nonce)
challenge_hash = xor(decrypted[0:16], key)
expected_crc = crc32(decrypted[0:12])
actual_crc = decrypted[12:16]
if expected_crc != actual_crc:
    return ERROR_INVALID_CRC
```

The `challenge_hash` is used in the response to prove the server could decrypt.

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
5. **MUST** include challenge even for unencrypted (Type 0) requests

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

## Related

- See `00_CRC32.md` for CRC32 algorithm details
- See `../02_Request_Headers/01_Header_32_Byte.md` for header construction
