# Response Header

## Size

Response headers are always 32 bytes, regardless of encryption type.

## Byte Map

```
Offset: 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
        RI SH SS CG UD UD EC EC RE SZ SZ SZ EX EX EX EX
        HS HS HS HS HS HS HS HS HS HS HS HS HS HS HS HS
```

## Field Reference

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 1 | RI | Responding RAIDA ID (0-24) |
| 1 | 1 | SH | Shard ID (0x00) |
| 2 | 1 | SS | Status Code |
| 3 | 1 | CG | Command Group (echoed from request) |
| 4-5 | 2 | UD | UDP Frame Count (0x0001) |
| 6-7 | 2 | EC | Client Echo (from request bytes 30-31) |
| 8 | 1 | RE | Reserved (0x00) |
| 9-11 | 3 | SZ | Body Size in bytes (big-endian) |
| 12-15 | 4 | EX | Execution Time in microseconds (big-endian) |
| 16-31 | 16 | HS | Signature |

## Signature Field (Bytes 16-31)

The signature calculation differs by encryption type:

| Type | Signature Calculation |
|------|----------------------|
| 0 | signature = sent_challenge |
| 1 | signature = sent_challenge XOR encryption_an |
| 2 | signature = sent_challenge XOR encryption_an |

**Type 0 (Unencrypted):**
The signature is simply the 16-byte challenge that was sent in the request body. No XOR operation.

**Types 1 & 2 (Encrypted):**
The signature is the challenge XOR'd with the encryption key (token/locker AN).

## Response Body

**Important:** The response body does NOT contain a challenge/CRC structure like the request body. The server's "answer" to the client's challenge is the signature in the response header (bytes 16-31), not in the body.

The response body format differs by encryption type:

| Type | Body Format |
|------|-------------|
| 0 | [Plaintext Payload][0x3E3E] |
| 1 | [AES-CTR Encrypted Payload][0x3E3E] |
| 2 | [AES-CTR Encrypted Payload][0x3E3E] |

```
Request Body:  [Challenge 16B][Command Data][0x3E3E]  <- Has challenge
Response Body: [Command Output][0x3E3E]               <- NO challenge
```

**Key Points:**
- The terminator (0x3E3E) is NEVER encrypted
- For Types 1 & 2, the same key and nonce from the request are used to encrypt the response
- The response body only contains the command output, no challenge/CRC prefix

## Parsing Example

```python
def parse_response_header(data: bytes) -> dict:
    if len(data) < 32:
        raise ValueError("Response too short")

    return {
        'raida_id': data[0],
        'shard_id': data[1],
        'status': data[2],
        'command_group': data[3],
        'udp_frame': int.from_bytes(data[4:6], 'big'),
        'echo': data[6:8],
        'body_size': int.from_bytes(data[9:12], 'big'),
        'exec_time_us': int.from_bytes(data[12:16], 'big'),
        'signature': data[16:32]
    }
```

## Signature Validation

```python
def validate_signature_type0(response_signature: bytes,
                             sent_challenge: bytes) -> bool:
    """Validate signature for Type 0 (unencrypted)."""
    return response_signature == sent_challenge

def validate_signature_type1_2(response_signature: bytes,
                               sent_challenge: bytes,
                               encryption_an: bytes) -> bool:
    """Validate signature for Types 1 and 2 (encrypted)."""
    expected = bytes(a ^ b for a, b in zip(sent_challenge, encryption_an))
    return response_signature == expected
```

## Response Body Decryption

For Types 1 and 2, the client must decrypt the response body:

```python
def decrypt_response_body(encrypted_body: bytes,
                          encryption_an: bytes,
                          nonce_8bytes: bytes) -> bytes:
    """
    Decrypt response body for Types 1 and 2.

    Args:
        encrypted_body: Encrypted payload (excluding terminator)
        encryption_an: 16-byte AN used for encryption
        nonce_8bytes: Same 8-byte nonce used in request header

    Returns:
        Decrypted payload
    """
    # Build 16-byte nonce (same as server uses)
    nonce_16 = nonce_8bytes + b'\x00' * 8

    # AES-CTR decrypt (same as encrypt)
    return aes_ctr_decrypt(encrypted_body, encryption_an, nonce_16)
```

## Complete Response Handling Example

```python
def handle_response(response: bytes, encryption_type: int,
                    sent_challenge: bytes, encryption_an: bytes = None,
                    nonce: bytes = None) -> dict:
    """
    Parse and validate a RAIDA response.

    Returns dict with parsed data or raises exception on error.
    """
    # Parse header
    header = parse_response_header(response[:32])

    # Validate signature
    if encryption_type == 0:
        valid = validate_signature_type0(header['signature'], sent_challenge)
    else:
        valid = validate_signature_type1_2(header['signature'],
                                           sent_challenge, encryption_an)

    if not valid:
        raise ValueError("Invalid response signature")

    # Extract body (if present)
    body_size = header['body_size']
    if body_size > 0:
        body_start = 32
        body_end = 32 + body_size - 2  # Exclude terminator
        encrypted_body = response[body_start:body_end]

        # Decrypt if needed
        if encryption_type in (1, 2):
            body = decrypt_response_body(encrypted_body, encryption_an, nonce)
        else:
            body = encrypted_body

        header['body'] = body

    return header
```
