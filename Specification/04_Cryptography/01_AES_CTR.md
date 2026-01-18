# AES-CTR Implementation

## Algorithm

The RAIDA protocol uses AES-128 in CTR (Counter) mode.

## Parameters

| Parameter | Value |
|-----------|-------|
| Cipher | AES |
| Mode | CTR (Counter) |
| Block Size | 16 bytes (128 bits) |
| Key Size | 128 bits (16 bytes) |
| Nonce Size | 8 bytes (from header) + 8 bytes zeros |

## Counter Construction

The 16-byte IV for AES-CTR is constructed as:

```
IV = [8-byte nonce from header] || [8 zero bytes]
```

### From 32-byte Header (Types 1, 2)

```
Nonce bytes: Header[24:32] (bytes 24-31)
```

Note: Bytes 30-31 also serve as echo bytes (dual purpose).

### Server Internal Construction

```python
# Server constructs 16-byte nonce from 8-byte header nonce
nonce_16 = header[24:32] + b'\x00' * 8
```

## Encryption Process

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

def aes_ctr_encrypt(plaintext: bytes, key: bytes, nonce_8: bytes) -> bytes:
    """
    Encrypt using AES-128-CTR.

    Args:
        plaintext: Data to encrypt
        key: 16 bytes (AES-128 key)
        nonce_8: 8 bytes from header

    Returns:
        Ciphertext (same length as plaintext)
    """
    # Construct 16-byte IV: nonce + 8 zero bytes
    iv = nonce_8 + b'\x00' * 8

    cipher = Cipher(algorithms.AES(key), modes.CTR(iv))
    encryptor = cipher.encryptor()
    return encryptor.update(plaintext) + encryptor.finalize()

def aes_ctr_decrypt(ciphertext: bytes, key: bytes, nonce_8: bytes) -> bytes:
    """
    Decrypt using AES-128-CTR.
    (Same as encrypt - CTR mode is symmetric)
    """
    return aes_ctr_encrypt(ciphertext, key, nonce_8)
```

## What Gets Encrypted

For Types 1 and 2:
- The request body IS encrypted (challenge + command payload)
- The terminator (0x3E3E) is NOT encrypted

```
Request: [Header 32B][Encrypted Body][Terminator 0x3E3E]
                     ^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^
                     Encrypted        NOT encrypted
```

## Nonce Requirements

1. **Uniqueness**: MUST never reuse a nonce with the same key
2. **Generation**: Use cryptographically secure random generator
3. **Size**: 8 bytes in header (padded to 16 internally)

```python
import os

def generate_nonce() -> bytes:
    """Generate 8-byte nonce for header."""
    return os.urandom(8)
```

## Echo Byte Consideration

Since bytes 30-31 serve as both nonce bytes 6-7 AND echo bytes:

```python
def generate_nonce_with_echo(echo: bytes) -> bytes:
    """
    Generate 8-byte nonce with specific echo value in last 2 bytes.

    Args:
        echo: 2-byte client tracking value

    Returns:
        8-byte nonce with echo in bytes 6-7
    """
    random_part = os.urandom(6)  # Random bytes 0-5
    return random_part + echo[:2]  # Echo in bytes 6-7
```

## Common Mistake

**DON'T** reuse nonces:
```python
# WRONG - Security vulnerability
nonce = b'\x00' * 8  # Static nonce
for raida in range(25):
    encrypt(body, key, nonce)  # Same nonce reused!
```

**DO** generate fresh nonce per request:
```python
# CORRECT
for raida in range(25):
    nonce = os.urandom(8)
    encrypt(body, key, nonce)
```
