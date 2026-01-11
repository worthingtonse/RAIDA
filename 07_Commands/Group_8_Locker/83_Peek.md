# PEEK (Group 8, Code 83)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 8 |
| Command Code | 83 |
| Header Size | 32 bytes |
| Request Body | 50 bytes |
| Response Body | Token summary + terminator |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Views locker contents without removing tokens.
Used to verify locker contents before Remove.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16-31 | 16 | Locker ID | ID returned from Put |
| 32-47 | 16 | Locker Code | Access code |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

## Response

| Status | Body |
|--------|------|
| 250 (SUCCESS) | Token summary |
| 179 (LOCKER_EMPTY) | None |
| 242 (FAIL) | None (wrong code) |

### Success Response Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 1 | Token Count | Number of tokens in locker |
| 1+ | 5*N | Tokens | Token identifiers only |

### Token Identifier (5 bytes each)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | Denomination |
| 1-4 | 4 | Serial Number (BE) |

Note: ANs are NOT returned by Peek (security).

## Algorithm

### Build Request
```pseudocode
function build_peek_request(
    raida_id: int,
    locker_id: bytes,
    locker_code: bytes,
    key_token: Token  // For encryption
) -> bytes:
    body_len = 16 + 16 + 16 + 2

    header = build_header_32(
        raida_id = raida_id,
        command_group = 8,
        command_code = 83,
        encryption_type = 1,
        key_dn = key_token.denomination,
        key_sn = key_token.serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    body += locker_id
    body += locker_code

    padded = pad_body(body, encryption_type=1)
    key = key_token.an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

### Parse Response
```pseudocode
function parse_peek_response(response: bytes) -> list:
    header = parse_response_header(response)

    if header.status == 179:
        return []  // Locker empty or not found

    if header.status != 250:
        raise Error("Peek failed")

    body = response[32:-2]
    token_count = body[0]

    tokens = []
    for i in range(token_count):
        offset = 1 + (i * 5)
        tokens.append({
            'denomination': body[offset],
            'serial_number': int.from_bytes(body[offset+1:offset+5], 'big')
        })

    return tokens
```

## Use Cases

### 1. Verify Before Accept
```pseudocode
function verify_locker_contents(
    locker_id: bytes,
    locker_code: bytes,
    expected_value: int
) -> bool:
    tokens = peek_all_raida(locker_id, locker_code)

    // Calculate total value
    total = 0
    for token in tokens:
        total += DENOMINATION_VALUES[token.denomination]

    return total >= expected_value
```

### 2. Check Locker Status
```pseudocode
function locker_exists(locker_id: bytes, locker_code: bytes) -> bool:
    for raida_id in range(25):
        result = peek(raida_id, locker_id, locker_code)
        if result is not None:
            return True
    return False
```

## Security Notes

1. **ANs not exposed**: Peek only returns DN/SN, not ownership secrets
2. **Code required**: Must know locker code to peek
3. **Non-destructive**: Tokens remain in locker after Peek

## Common Mistakes

**DON'T** assume Peek returns ANs:
```python
# WRONG - Peek doesn't return ANs
tokens = peek(locker_id, code)
for token in tokens:
    use(token.an)  # Error! No AN in peek response
```

**DO** use Remove to get full tokens:
```python
# CORRECT
summary = peek(locker_id, code)  # Check contents first
if summary_looks_good(summary):
    tokens = remove(locker_id, code)  # Get full tokens with ANs
```
