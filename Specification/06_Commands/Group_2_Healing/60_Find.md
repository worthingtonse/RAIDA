# FIND (Group 2, Code 60)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 2 |
| Command Code | 60 |
| Header Size | 32 bytes |
| Request Body | Variable |
| Response Body | Find result + terminator |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Determines whether a RAIDA knows a token by AN, PAN, or neither.
Used during healing to understand token state across network.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16 | 1 | Denomination | Token denomination |
| 17-20 | 4 | Serial Number | Token SN (BE) |
| 21-36 | 16 | AN | Current Authenticity Number |
| 37-52 | 16 | PAN | Proposed AN (if known) |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

## Response

| Status | Meaning |
|--------|---------|
| 208 (FIND_NEITHER) | Neither AN nor PAN found |
| 209 (FIND_ALL_AN) | AN matched |
| 210 (FIND_ALL_PAN) | PAN matched |
| 211 (FIND_MIXED) | Bitfield in body |

### Status Code Details

- **208**: Token exists but with different AN (or doesn't exist)
- **209**: Token has the AN we sent (current owner)
- **210**: Token has the PAN we sent (transfer in progress)
- **211**: Multiple tokens with mixed results (batch)

## Algorithm

### Build Request
```pseudocode
function build_find_request(
    raida_id: int,
    token: Token,
    pan: bytes = None
) -> bytes:
    if pan is None:
        pan = bytes(16)  # Zero PAN if unknown

    body_len = 16 + 1 + 4 + 16 + 16 + 2
    header = build_header_32(
        raida_id = raida_id,
        command_group = 2,
        command_code = 60,
        encryption_type = 1,
        key_dn = token.denomination,
        key_sn = token.serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    body += bytes([token.denomination])
    body += token.serial_number.to_bytes(4, 'big')
    body += token.an[raida_id]
    body += pan

    padded = pad_body(body, encryption_type=1)
    key = token.an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

### Interpret Results
```pseudocode
function interpret_find_result(status: int) -> str:
    if status == 208:
        return "unknown"      // RAIDA doesn't recognize token
    elif status == 209:
        return "has_an"       // RAIDA has current AN
    elif status == 210:
        return "has_pan"      // RAIDA has PAN (transfer started)
    elif status == 211:
        return "mixed"        // Check bitfield
```

## Use Cases

### 1. Diagnosing Fractured Token
```pseudocode
function diagnose_token(token: Token) -> dict:
    results = {}
    for raida_id in range(25):
        status = find(raida_id, token)
        results[raida_id] = interpret_find_result(status)

    return {
        'has_an': count(results, 'has_an'),
        'has_pan': count(results, 'has_pan'),
        'unknown': count(results, 'unknown')
    }
```

### 2. Recovering from Failed Transfer
```pseudocode
function recover_failed_transfer(token: Token, pan: bytes):
    // Check which RAIDA have old AN vs new PAN
    for raida_id in range(25):
        status = find(raida_id, token, pan)

        if status == 209:
            // Still has old AN - transfer didn't complete here
            // Can retry POWN
            pown(raida_id, token, pan)

        elif status == 210:
            // Has new PAN - transfer completed here
            // Update local token
            token.an[raida_id] = pan
```

## Common Mistakes

**DON'T** use Find when Detect is sufficient:
```python
# WRONG - Detect is simpler for just checking ownership
find(raida_id, token, empty_pan)
```

**DO** use Find when you need AN vs PAN distinction:
```python
# CORRECT - Need to know if transfer partially completed
find(raida_id, token, attempted_pan)
```
