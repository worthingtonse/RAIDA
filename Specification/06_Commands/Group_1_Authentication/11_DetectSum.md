# DETECT SUM (Group 1, Code 11)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 1 |
| Command Code | 11 |
| Header Size | 32 bytes |
| Request Body | 18 + (5 * N) + 16 bytes |
| Response Body | Conditional (bitfield if mixed) |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Optimized Detect command using checksum instead of full ANs.
Reduces bandwidth when authenticating many tokens of same denomination.

## How It Works

Instead of sending 16-byte AN per token, client sends:
1. Token identifiers (DN + SN = 5 bytes each)
2. Single 16-byte checksum of all ANs

Server computes same checksum and compares.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16+ | 5*N | Tokens | N token identifiers |
| +16 | 16 | AN Checksum | MD5 of concatenated ANs |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

### Token Identifier (5 bytes each)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | Denomination |
| 1-4 | 4 | Serial Number (BE) |

### Checksum Calculation

```pseudocode
function calculate_an_checksum(tokens: list, raida_id: int) -> bytes:
    // Concatenate all ANs in order
    all_ans = b''
    for token in tokens:
        all_ans += token.an[raida_id]

    // MD5 hash of concatenated ANs
    return md5(all_ans)
```

## Response

| Status | Body |
|--------|------|
| 241 (ALL_PASS) | None |
| 242 (ALL_FAIL) | None |
| 243 (MIXED) | Bitfield + terminator |

## Algorithm

### Build Request
```pseudocode
function build_detect_sum_request(tokens: list, raida_id: int) -> bytes:
    // Calculate checksum
    checksum = calculate_an_checksum(tokens, raida_id)

    body_len = 16 + (5 * len(tokens)) + 16 + 2
    header = build_header_32(
        raida_id = raida_id,
        command_group = 1,
        command_code = 11,
        encryption_type = 1,
        key_dn = tokens[0].denomination,
        key_sn = tokens[0].serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    for token in tokens:
        body += bytes([token.denomination])
        body += token.serial_number.to_bytes(4, 'big')
    body += checksum

    padded = pad_body(body, encryption_type=1)
    key = tokens[0].an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

## Bandwidth Comparison

| Tokens | Standard Detect | DetectSum | Savings |
|--------|-----------------|-----------|---------|
| 10 | 210 bytes | 66 bytes | 69% |
| 50 | 1050 bytes | 266 bytes | 75% |
| 100 | 2100 bytes | 516 bytes | 75% |

Formula:
- Standard: 21 * N bytes (DN + SN + AN per token)
- DetectSum: 5 * N + 16 bytes (DN + SN per token + single checksum)

## Limitations

1. **All-or-nothing on checksum mismatch**: If any AN is wrong, checksum fails
2. **Cannot identify which token failed**: Must fall back to standard Detect
3. **Same denomination recommended**: Mixed denominations complicate key selection

## Use Cases

1. **Wallet verification**: Quick check that all tokens are authentic
2. **Pre-transfer validation**: Verify before sending large batch
3. **Periodic health check**: Confirm token inventory

## Common Mistakes

**DON'T** use DetectSum when you need per-token results:
```python
# WRONG - Can't identify bad token
result = detect_sum(tokens)
if result == FAIL:
    # Which token is bad? Unknown!
```

**DO** fall back to standard Detect on failure:
```python
# CORRECT
result = detect_sum(tokens)
if result == ALL_PASS:
    return [True] * len(tokens)
else:
    # Fall back to individual detection
    return [detect(token) for token in tokens]
```
