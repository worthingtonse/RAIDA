# POWN SUM (Group 1, Code 21)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 1 |
| Command Code | 21 |
| Header Size | 32 bytes |
| Request Body | 18 + (21 * N) + 16 bytes |
| Response Body | Conditional (bitfield if mixed) |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Optimized POWN command using checksum for current ANs.
Reduces bandwidth while still sending full PANs.

## How It Works

1. Send token identifiers + PANs (21 bytes each)
2. Send single checksum of all current ANs
3. Server verifies checksum, then applies PANs

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16+ | 21*N | Tokens | N token records |
| +16 | 16 | AN Checksum | MD5 of current ANs |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

### Token Record (21 bytes each)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | Denomination |
| 1-4 | 4 | Serial Number (BE) |
| 5-20 | 16 | Proposed AN (PAN) |

Note: Current AN is NOT sent per-token; only checksum is sent.

### Checksum Calculation

```pseudocode
function calculate_an_checksum(tokens: list, raida_id: int) -> bytes:
    all_ans = b''
    for token in tokens:
        all_ans += token.an[raida_id]  // Current AN
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
function build_pown_sum_request(
    tokens: list,
    pans: list,
    raida_id: int
) -> bytes:
    checksum = calculate_an_checksum(tokens, raida_id)

    body_len = 16 + (21 * len(tokens)) + 16 + 2
    header = build_header_32(
        raida_id = raida_id,
        command_group = 1,
        command_code = 21,
        encryption_type = 1,
        key_dn = tokens[0].denomination,
        key_sn = tokens[0].serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    for i, token in enumerate(tokens):
        body += bytes([token.denomination])
        body += token.serial_number.to_bytes(4, 'big')
        body += pans[i][raida_id]  // New AN
    body += checksum

    padded = pad_body(body, encryption_type=1)
    key = tokens[0].an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

## Bandwidth Comparison

| Tokens | Standard POWN | PownSum | Savings |
|--------|---------------|---------|---------|
| 10 | 370 bytes | 226 bytes | 39% |
| 50 | 1850 bytes | 1066 bytes | 42% |
| 100 | 3700 bytes | 2116 bytes | 43% |

Formula:
- Standard: 37 * N bytes (DN + SN + AN + PAN per token)
- PownSum: 21 * N + 16 bytes (DN + SN + PAN per token + checksum)

## Limitations

1. **All-or-nothing verification**: Checksum must match for any POWN to occur
2. **Atomic batch**: Either all tokens POWN or none do
3. **No partial recovery**: On failure, must retry entire batch

## When to Use

| Scenario | Use Standard POWN | Use PownSum |
|----------|-------------------|-------------|
| Single token | ✓ | |
| Mixed reliability | ✓ | |
| Known-good batch | | ✓ |
| Bandwidth limited | | ✓ |

## Common Mistakes

**DON'T** use PownSum for untested tokens:
```python
# WRONG - If any AN is bad, all fail
pown_sum(unverified_tokens, pans)
```

**DO** verify first with DetectSum:
```python
# CORRECT
if detect_sum(tokens) == ALL_PASS:
    pown_sum(tokens, pans)
else:
    # Fall back to individual POWN
    for token, pan in zip(tokens, pans):
        pown(token, pan)
```
