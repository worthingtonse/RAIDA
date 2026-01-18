# PUT (Group 8, Code 82)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 8 |
| Command Code | 82 |
| Header Size | 32 bytes |
| Request Body | Variable (large) |
| Response Body | Locker ID + terminator |
| Encryption | Required |
| Transport | TCP recommended |

## Purpose

Stores tokens in a server-side locker for later retrieval.
Used for offline transfers and escrow scenarios.

## Concept

```
Sender                    RAIDA Locker                Recipient
   |                           |                           |
   |-- Put(tokens, code) ----->|                           |
   |<-- Locker ID -------------|                           |
   |                           |                           |
   |--- Share ID + code ---------------------------------->|
   |                           |                           |
   |                           |<-- Remove(ID, code) ------|
   |                           |-- tokens ---------------->|
```

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16-31 | 16 | Locker Code | Access code for retrieval |
| 32 | 1 | Token Count | Number of tokens (N) |
| 33+ | 37*N | Tokens | Token records with PANs |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

### Token Record (37 bytes each)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | Denomination |
| 1-4 | 4 | Serial Number (BE) |
| 5-20 | 16 | Current AN |
| 21-36 | 16 | PAN (for locker) |

## Response

| Status | Body |
|--------|------|
| 250 (SUCCESS) | 16-byte Locker ID |
| 242 (FAIL) | None |

## Algorithm

### Build Request
```pseudocode
function build_put_request(
    raida_id: int,
    tokens: list,
    locker_code: bytes,  // Recipient will need this
    pans: list           // New ANs for locker ownership
) -> bytes:
    token_count = len(tokens)
    body_len = 16 + 16 + 1 + (37 * token_count) + 2

    header = build_header_32(
        raida_id = raida_id,
        command_group = 8,
        command_code = 82,
        encryption_type = 1,
        key_dn = tokens[0].denomination,
        key_sn = tokens[0].serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    body += locker_code
    body += bytes([token_count])

    for i, token in enumerate(tokens):
        body += bytes([token.denomination])
        body += token.serial_number.to_bytes(4, 'big')
        body += token.an[raida_id]
        body += pans[i][raida_id]

    padded = pad_body(body, encryption_type=1)
    key = tokens[0].an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

### Parse Response
```pseudocode
function parse_put_response(response: bytes) -> bytes:
    header = parse_response_header(response)

    if header.status == 250:
        locker_id = response[32:48]
        return locker_id
    else:
        return None
```

### Complete Put Workflow
```pseudocode
function put_tokens_in_locker(
    tokens: list,
    locker_code: bytes
) -> dict:
    // Generate PANs for locker
    pans = [[os.urandom(16) for _ in range(25)] for _ in tokens]

    // Send to all 25 RAIDA
    locker_ids = []
    for raida_id in range(25):
        result = put(raida_id, tokens, locker_code, pans)
        if result:
            locker_ids.append((raida_id, result))

    // Need consensus
    if len(locker_ids) >= 13:
        return {
            'success': True,
            'locker_ids': locker_ids,
            'code': locker_code
        }
    else:
        return {'success': False}
```

## Locker Code

The locker code is a 16-byte secret that:
- Sender generates randomly
- Sender shares with recipient out-of-band
- Recipient uses to retrieve tokens

```python
locker_code = os.urandom(16)
# Share locker_code + locker_id with recipient securely
```

## Transport Selection

Put typically requires TCP due to payload size:

| Tokens | Body Size | Transport |
|--------|-----------|-----------|
| 1-35 | < 1400 bytes | UDP ok |
| 36+ | > 1400 bytes | TCP required |

## Common Mistakes

**DON'T** reuse locker codes:
```python
# WRONG - Security risk
LOCKER_CODE = b'fixed_code_1234!'  # Reused!
put(tokens, LOCKER_CODE)
```

**DO** generate unique code per locker:
```python
# CORRECT
locker_code = os.urandom(16)
put(tokens, locker_code)
```

**DON'T** share locker info insecurely:
```python
# WRONG
email_plaintext(locker_id, locker_code)  # Interceptable!
```

**DO** use secure channel for sharing:
```python
# CORRECT
encrypted_share(locker_id, locker_code)
# Or: in-person, encrypted messaging, etc.
```
