# DETECT (Group 1, Code 10)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 1 |
| Command Code | 10 |
| Header Size | 32 bytes |
| Request Body | 18 + (21 * N) bytes |
| Response Body | Conditional (bitfield if mixed) |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Verifies token authenticity without changing the AN.
Used for health checks and pre-transfer validation.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16+ | 21*N | Tokens | N token records |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

### Token Record (21 bytes each)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | Denomination |
| 1-4 | 4 | Serial Number (BE) |
| 5-20 | 16 | Authenticity Number |

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
function build_detect_request(tokens: list, raida_id: int) -> bytes:
    // Step 1: Build header
    body_len = 16 + (21 * len(tokens)) + 2  // challenge + tokens + term
    header = build_header_32(
        raida_id = raida_id,
        command_group = 1,
        command_code = 10,
        encryption_type = 1,
        key_dn = tokens[0].denomination,
        key_sn = tokens[0].serial_number,
        body_length = body_len
    )

    // Step 2: Build body
    body = generate_challenge()
    for token in tokens:
        body += bytes([token.denomination])
        body += token.serial_number.to_bytes(4, 'big')
        body += token.an[raida_id]

    // Step 3: Pad and encrypt
    padded = pad_body(body, encryption_type=1)
    key = tokens[0].an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

### Parse Response
```pseudocode
function parse_detect_response(response: bytes, token_count: int) -> list:
    header = parse_response_header(response)

    if header.status == 241:
        return [True] * token_count
    elif header.status == 242:
        return [False] * token_count
    elif header.status == 243:
        bitfield = response[32:-2]
        return parse_bitfield(bitfield, token_count)
```

## Example

### Request with 2 tokens
```
Body before encryption:
[16B challenge]
01 00 01 8F 3D [16B AN for token 1]  // DN=1, SN=102205
01 00 01 8F 3E [16B AN for token 2]  // DN=1, SN=102206
3E 3E
```

### Response (ALL_PASS)
```
Header: ... status=0xF1 (241) ...
No body
```

### Response (MIXED, 2 tokens)
```
Header: ... status=0xF3 (243) ...
Body: C0 3E 3E
      ^^ bitfield: 11000000 = token 0 pass, token 1 pass
         (padded with zeros)
```

## Test Vectors

### Test 1: Single Token Pass
```json
{
  "input": {
    "tokens": [{"dn": 1, "sn": 102205, "an": "valid_an"}]
  },
  "expected": {"status": 241}
}
```

### Test 2: Single Token Fail
```json
{
  "input": {
    "tokens": [{"dn": 1, "sn": 102205, "an": "wrong_an"}]
  },
  "expected": {"status": 242}
}
```

## Common Mistakes

**DON'T** use the same AN for all RAIDA:
```python
# WRONG
an = token.an[0]
for raida in range(25):
    send_detect(raida, an)  # Same AN!
```

**DO** use RAIDA-specific AN:
```python
# CORRECT
for raida in range(25):
    an = token.an[raida]  # Different AN per RAIDA
    send_detect(raida, an)
```
