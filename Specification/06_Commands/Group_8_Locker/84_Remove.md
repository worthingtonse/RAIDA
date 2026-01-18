# REMOVE (Group 8, Code 84)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 8 |
| Command Code | 84 |
| Header Size | 32 bytes |
| Request Body | 50 + (16 * N) bytes |
| Response Body | Full tokens + terminator |
| Encryption | Required |
| Transport | TCP recommended |

## Purpose

Retrieves and removes tokens from a locker.
Transfers ownership to the recipient's PANs.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16-31 | 16 | Locker ID | ID from Put response |
| 32-47 | 16 | Locker Code | Access code |
| 48 | 1 | Token Count | N tokens to retrieve |
| 49+ | 16*N | PANs | New ANs for recipient |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

## Response

| Status | Body |
|--------|------|
| 250 (SUCCESS) | Token records with new ANs |
| 179 (LOCKER_EMPTY) | None |
| 242 (FAIL) | None (wrong code) |

### Success Response Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 1 | Token Count | N |
| 1+ | 21*N | Tokens | Full token records |

### Token Record in Response (21 bytes each)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | Denomination |
| 1-4 | 4 | Serial Number (BE) |
| 5-20 | 16 | New AN (from PAN sent) |

## Algorithm

### Build Request
```pseudocode
function build_remove_request(
    raida_id: int,
    locker_id: bytes,
    locker_code: bytes,
    token_count: int,
    pans: list,  // New ANs for recipient
    key_token: Token
) -> bytes:
    body_len = 16 + 16 + 16 + 1 + (16 * token_count) + 2

    header = build_header_32(
        raida_id = raida_id,
        command_group = 8,
        command_code = 84,
        encryption_type = 1,
        key_dn = key_token.denomination,
        key_sn = key_token.serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    body += locker_id
    body += locker_code
    body += bytes([token_count])
    for pan in pans:
        body += pan[raida_id]

    padded = pad_body(body, encryption_type=1)
    key = key_token.an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

### Parse Response
```pseudocode
function parse_remove_response(response: bytes) -> list:
    header = parse_response_header(response)

    if header.status == 179:
        return []  // Locker empty

    if header.status != 250:
        raise Error("Remove failed")

    body = response[32:-2]
    token_count = body[0]

    tokens = []
    for i in range(token_count):
        offset = 1 + (i * 21)
        tokens.append({
            'denomination': body[offset],
            'serial_number': int.from_bytes(body[offset+1:offset+5], 'big'),
            'an': body[offset+5:offset+21]
        })

    return tokens
```

### Complete Remove Workflow
```pseudocode
function remove_from_locker(
    locker_id: bytes,
    locker_code: bytes,
    expected_count: int
) -> list:
    // Step 1: Peek to verify contents
    summary = peek_all_raida(locker_id, locker_code)
    if len(summary) != expected_count:
        raise Error("Unexpected token count")

    // Step 2: Generate PANs for my ownership
    pans = [[os.urandom(16) for _ in range(25)] for _ in range(expected_count)]

    // Step 3: Remove from all RAIDA
    all_tokens = []
    for raida_id in range(25):
        tokens = remove(raida_id, locker_id, locker_code, expected_count, pans)
        for i, token in enumerate(tokens):
            if len(all_tokens) <= i:
                all_tokens.append({
                    'denomination': token['denomination'],
                    'serial_number': token['serial_number'],
                    'ans': [None] * 25
                })
            all_tokens[i]['ans'][raida_id] = pans[i][raida_id]

    return all_tokens
```

## Locker Lifecycle

```
1. Sender: Put(tokens, code) --> locker_id
2. Sender: Share locker_id + code with Recipient
3. Recipient: Peek(locker_id, code) --> verify contents
4. Recipient: Remove(locker_id, code, PANs) --> own tokens
5. Locker: Destroyed (empty)
```

## Security Considerations

1. **One-time retrieval**: Remove empties the locker
2. **Code required**: Must know locker code
3. **POWN occurs**: Tokens are POWNed to recipient's PANs
4. **Atomic per-RAIDA**: Each RAIDA handles independently

## Common Mistakes

**DON'T** forget to generate PANs:
```python
# WRONG - No PANs means no ownership!
remove(locker_id, code, count, pans=[])
```

**DO** generate unique PANs per token per RAIDA:
```python
# CORRECT
pans = [[os.urandom(16) for _ in range(25)] for _ in range(count)]
remove(locker_id, code, count, pans)
```

**DON'T** skip peek verification:
```python
# RISKY - Don't know what you're getting
tokens = remove(locker_id, code, pans)
```

**DO** verify with Peek first:
```python
# CORRECT
summary = peek(locker_id, code)
if validate_summary(summary):
    tokens = remove(locker_id, code, len(summary), pans)
```
