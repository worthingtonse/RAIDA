# POWN (Group 1, Code 20)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 1 |
| Command Code | 20 |
| Header Size | 32 bytes |
| Request Body | 18 + (37 * N) bytes |
| Response Body | Conditional (bitfield if mixed) |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Changes token ownership by replacing the current AN with a new PAN.
This is the primary transfer mechanism for tokens.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16+ | 37*N | Tokens | N token records |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

### Token Record (37 bytes each)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | Denomination |
| 1-4 | 4 | Serial Number (BE) |
| 5-20 | 16 | Current AN |
| 21-36 | 16 | Proposed AN (PAN) |

## Response

| Status | Body |
|--------|------|
| 241 (ALL_PASS) | None |
| 242 (ALL_FAIL) | None |
| 243 (MIXED) | Bitfield + terminator |

### Bitfield Format

- 1 bit per token (1=pass, 0=fail)
- MSB of byte 0 = token 0
- Padded to full bytes with 0 bits

## Algorithm

### Build Request
```pseudocode
function build_pown_request(tokens: list, pans: list, raida_id: int) -> bytes:
    // Step 1: Build header
    body_len = 16 + (37 * len(tokens)) + 2
    header = build_header_32(
        raida_id = raida_id,
        command_group = 1,
        command_code = 20,
        encryption_type = 1,
        key_dn = tokens[0].denomination,
        key_sn = tokens[0].serial_number,
        body_length = body_len
    )

    // Step 2: Build body
    body = generate_challenge()
    for i, token in enumerate(tokens):
        body += bytes([token.denomination])
        body += token.serial_number.to_bytes(4, 'big')
        body += token.an[raida_id]
        body += pans[i][raida_id]  // New AN for this RAIDA

    // Step 3: Pad and encrypt
    padded = pad_body(body, encryption_type=1)
    key = tokens[0].an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

### Parse Response
```pseudocode
function parse_pown_response(response: bytes, token_count: int) -> list:
    header = parse_response_header(response)

    if header.status == 241:
        return [True] * token_count
    elif header.status == 242:
        return [False] * token_count
    elif header.status == 243:
        bitfield = response[32:-2]
        return parse_bitfield(bitfield, token_count)
```

### After Successful POWN
```pseudocode
function update_token_after_pown(token, pan, raida_id: int, success: bool):
    if success:
        // Replace old AN with PAN
        token.an[raida_id] = pan[raida_id]
    // If failed, keep old AN (may need healing later)
```

## Example

### Request with 1 token
```
Body before encryption:
[16B challenge]
01                  // Denomination = 1
00 01 8F 3D         // SN = 102205 (big-endian)
[16B current AN]    // What RAIDA knows
[16B proposed AN]   // What we want RAIDA to store
3E 3E
```

### Response (ALL_PASS)
```
Header: ... status=0xF1 (241) ...
No body
```

## Consensus Requirements

After sending POWN to all 25 RAIDA:
- Count successes (status 241, or bit=1 in mixed)
- If >= 13 RAIDA succeed: Token is transferred
- If < 13 succeed: Token may be fractured (needs healing)

## Test Vectors

### Test 1: Single Token POWN
```json
{
  "input": {
    "token": {"dn": 1, "sn": 102205, "an": "current_an"},
    "pan": "new_proposed_an"
  },
  "expected": {"status": 241}
}
```

### Test 2: Invalid AN POWN
```json
{
  "input": {
    "token": {"dn": 1, "sn": 102205, "an": "wrong_an"},
    "pan": "new_proposed_an"
  },
  "expected": {"status": 242}
}
```

## Common Mistakes

**DON'T** reuse the same PAN for all RAIDA:
```python
# WRONG
pan = os.urandom(16)
for raida in range(25):
    send_pown(raida, current_an[raida], pan)  # Same PAN!
```

**DO** generate unique PAN per RAIDA:
```python
# CORRECT
pans = [os.urandom(16) for _ in range(25)]
for raida in range(25):
    send_pown(raida, current_an[raida], pans[raida])
```

**DON'T** update local token before consensus:
```python
# WRONG
for raida in range(25):
    result = send_pown(raida, an, pan)
    if result:
        token.an[raida] = pan[raida]  # Too early!
```

**DO** wait for consensus before updating:
```python
# CORRECT
results = [send_pown(raida, an, pan) for raida in range(25)]
successes = sum(results)
if successes >= 13:
    for raida, success in enumerate(results):
        if success:
            token.an[raida] = pan[raida]
```
