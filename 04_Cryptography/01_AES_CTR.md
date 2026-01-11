# AES-CTR Implementation

## Algorithm

The RAIDA protocol uses AES in CTR (Counter) mode.

## Parameters

| Parameter | Value |
|-----------|-------|
| Cipher | AES |
| Mode | CTR (Counter) |
| Block Size | 16 bytes (128 bits) |
| Key Size | 128 or 256 bits (depends on EN type) |
| Nonce Size | 8 bytes (from header) + 8 bytes counter |

## Counter Construction

The 16-byte IV for AES-CTR is constructed as:

```
IV = [8-byte nonce from header] || [8-byte counter starting at 0]
```

### From 32-byte Header (Types 1, 2, 3)
```
Nonce bytes: Header[24:32] (bytes 24-31)
```

### From 48-byte Header (Types 4, 5, 7)
```
Nonce bytes: Header[29:37] or as specified per type
```

## Encryption Process

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

def aes_ctr_encrypt(plaintext: bytes, key: bytes, nonce: bytes) -> bytes:
    """
    Encrypt using AES-CTR.

    Args:
        plaintext: Data to encrypt (padded to block size)
        key: 16 bytes (AES-128) or 32 bytes (AES-256)
        nonce: 8 bytes from header

    Returns:
        Ciphertext (same length as plaintext)
    """
    # Construct 16-byte IV: nonce + 8 zero bytes for counter
    iv = nonce + b'\x00' * 8

    cipher = Cipher(algorithms.AES(key), modes.CTR(iv))
    encryptor = cipher.encryptor()
    return encryptor.update(plaintext) + encryptor.finalize()

def aes_ctr_decrypt(ciphertext: bytes, key: bytes, nonce: bytes) -> bytes:
    """
    Decrypt using AES-CTR.
    (Same as encrypt - CTR mode is symmetric)
    """
    return aes_ctr_encrypt(ciphertext, key, nonce)
```

## Nonce Requirements

1. **Uniqueness**: MUST never reuse a nonce with the same key
2. **Generation**: Use cryptographically secure random generator
3. **Size**: 8 bytes

```python
import os

def generate_nonce() -> bytes:
    return os.urandom(8)
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
