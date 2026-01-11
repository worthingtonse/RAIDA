# Core Algorithms

## Overview

This section provides standalone algorithm implementations for key RAIDA operations.

## Algorithm Index

| Algorithm | Purpose | Document |
|-----------|---------|----------|
| CRC32 | Challenge validation | `01_CRC32.md` |
| AES-CTR | Packet encryption | `02_AES_CTR.md` |
| Bitfield | Parse mixed results | `03_Bitfield.md` |
| Consensus | Determine outcome | `04_Consensus.md` |

## CRC32 Algorithm

```python
import zlib
import struct

def calculate_crc32(data: bytes) -> bytes:
    """
    Calculate CRC32 checksum (zlib compatible).

    Args:
        data: Input bytes (typically 12 random bytes)

    Returns:
        4 bytes CRC32 in big-endian order
    """
    crc = zlib.crc32(data) & 0xFFFFFFFF
    return struct.pack('>I', crc)

def generate_challenge() -> bytes:
    """
    Generate 16-byte challenge (12 random + 4 CRC).

    Returns:
        16 bytes: random_data || CRC32(random_data)
    """
    import os
    random_data = os.urandom(12)
    crc = calculate_crc32(random_data)
    return random_data + crc

def validate_challenge(challenge: bytes) -> bool:
    """
    Validate challenge CRC.

    Args:
        challenge: 16-byte challenge

    Returns:
        True if CRC matches
    """
    if len(challenge) != 16:
        return False
    random_data = challenge[:12]
    provided_crc = challenge[12:16]
    expected_crc = calculate_crc32(random_data)
    return provided_crc == expected_crc
```

## AES-CTR Encryption

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend

def aes_ctr_encrypt(plaintext: bytes, key: bytes, nonce: bytes) -> bytes:
    """
    Encrypt using AES-CTR mode.

    Args:
        plaintext: Data to encrypt (should be padded)
        key: 16 bytes (AES-128) or 32 bytes (AES-256)
        nonce: 14 bytes from header

    Returns:
        Ciphertext (same length as plaintext)
    """
    # Build 16-byte counter block: nonce || zeros
    counter_block = nonce.ljust(16, b'\x00')

    cipher = Cipher(
        algorithms.AES(key),
        modes.CTR(counter_block),
        backend=default_backend()
    )
    encryptor = cipher.encryptor()
    return encryptor.update(plaintext) + encryptor.finalize()

def aes_ctr_decrypt(ciphertext: bytes, key: bytes, nonce: bytes) -> bytes:
    """
    Decrypt using AES-CTR mode.
    Note: CTR decryption is identical to encryption.
    """
    return aes_ctr_encrypt(ciphertext, key, nonce)

def pad_to_block(data: bytes, block_size: int) -> bytes:
    """
    Pad data with zeros to block boundary.

    Args:
        data: Input data
        block_size: 16 for AES-128, 32 for AES-256

    Returns:
        Padded data
    """
    remainder = len(data) % block_size
    if remainder == 0:
        return data
    padding_needed = block_size - remainder
    return data + (b'\x00' * padding_needed)
```

## Bitfield Parsing

```python
def parse_bitfield(bitfield: bytes, count: int) -> list:
    """
    Parse bitfield from MIXED (243) response.

    Args:
        bitfield: Bytes containing bit results
        count: Number of tokens/items

    Returns:
        List of booleans (True=pass, False=fail)
    """
    results = []
    for i in range(count):
        byte_index = i // 8
        bit_index = 7 - (i % 8)  # MSB first

        if byte_index < len(bitfield):
            bit = (bitfield[byte_index] >> bit_index) & 1
            results.append(bit == 1)
        else:
            results.append(False)

    return results

def build_bitfield(results: list) -> bytes:
    """
    Build bitfield from list of booleans.

    Args:
        results: List of booleans

    Returns:
        Packed bitfield bytes
    """
    byte_count = (len(results) + 7) // 8
    bitfield = bytearray(byte_count)

    for i, result in enumerate(results):
        if result:
            byte_index = i // 8
            bit_index = 7 - (i % 8)
            bitfield[byte_index] |= (1 << bit_index)

    return bytes(bitfield)
```

## Consensus Calculation

```python
def calculate_consensus(results: list, required: int = 13) -> dict:
    """
    Determine consensus from RAIDA results.

    Args:
        results: List of 25 results (True/False/None)
        required: Minimum for consensus (default 13)

    Returns:
        Dict with consensus info
    """
    passes = sum(1 for r in results if r is True)
    fails = sum(1 for r in results if r is False)
    errors = sum(1 for r in results if r is None)

    if passes >= required:
        status = 'consensus_pass'
    elif fails >= required:
        status = 'consensus_fail'
    elif passes > 0 and passes < required:
        status = 'fractured'
    else:
        status = 'error'

    return {
        'status': status,
        'passes': passes,
        'fails': fails,
        'errors': errors,
        'consensus_achieved': passes >= required
    }

def aggregate_token_results(
    all_results: list,  # 25 responses
    token_count: int
) -> list:
    """
    Aggregate per-token results from all RAIDA.

    Returns:
        List of consensus results per token
    """
    per_token = []

    for token_idx in range(token_count):
        token_results = []
        for raida_idx in range(25):
            response = all_results[raida_idx]
            if response is None:
                token_results.append(None)
            elif response['status'] == 241:  # ALL_PASS
                token_results.append(True)
            elif response['status'] == 242:  # ALL_FAIL
                token_results.append(False)
            elif response['status'] == 243:  # MIXED
                token_results.append(response['bitfield'][token_idx])

        per_token.append(calculate_consensus(token_results))

    return per_token
```

## Response Signature Validation

```python
def validate_response_signature(
    response_signature: bytes,
    sent_challenge: bytes,
    encryption_an: bytes
) -> bool:
    """
    Validate RAIDA response signature.

    The signature should equal: Challenge XOR Encryption_AN

    Args:
        response_signature: 16 bytes from response header
        sent_challenge: 16-byte challenge we sent
        encryption_an: AN used for encryption key

    Returns:
        True if signature is valid
    """
    expected = bytes(a ^ b for a, b in zip(sent_challenge, encryption_an))
    return response_signature == expected
```
