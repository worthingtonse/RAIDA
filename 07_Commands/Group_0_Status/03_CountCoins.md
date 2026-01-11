# COUNT COINS (Group 0, Code 3)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 0 |
| Command Code | 3 |
| Header Size | 32 bytes |
| Request Body | 18 bytes |
| Response Body | Coin counts + terminator |
| Encryption | Optional |
| Transport | UDP preferred |

## Purpose

Returns the count of tokens per denomination on this RAIDA.
Used for network-wide inventory and auditing.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16-17 | 2 | Terminator | 0x3E 0x3E |

## Response

| Status | Body |
|--------|------|
| 250 (SUCCESS) | Denomination counts + terminator |

### Count Structure

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-3 | 4 | DN 0 Count | Quarter tokens (BE) |
| 4-7 | 4 | DN 1 Count | 1-unit tokens (BE) |
| 8-11 | 4 | DN 2 Count | 5-unit tokens (BE) |
| 12-15 | 4 | DN 3 Count | 25-unit tokens (BE) |
| 16-19 | 4 | DN 4 Count | 100-unit tokens (BE) |
| 20-23 | 4 | DN 5 Count | 250-unit tokens (BE) |
| 24-27 | 4 | DN 6 Count | 1000-unit tokens (BE) |

## Algorithm

### Build Request
```pseudocode
function build_count_request(raida_id: int) -> bytes:
    header = build_header_32(
        raida_id = raida_id,
        command_group = 0,
        command_code = 3,
        encryption_type = 0,
        body_length = 18
    )

    challenge = generate_challenge()
    body = challenge + b'\x3E\x3E'

    return header + body
```

### Parse Response
```pseudocode
function parse_count_response(response: bytes) -> dict:
    header = parse_response_header(response)

    if header.status != 250:
        raise Error("Count request failed")

    body = response[32:-2]
    counts = {}
    for dn in range(7):
        offset = dn * 4
        counts[dn] = int.from_bytes(body[offset:offset+4], 'big')

    return counts
```

### Calculate Total Value
```pseudocode
DENOMINATION_VALUES = [0.25, 1, 5, 25, 100, 250, 1000]

function calculate_total_value(counts: dict) -> float:
    total = 0
    for dn, count in counts.items():
        total += count * DENOMINATION_VALUES[dn]
    return total
```

## Example

### Response Body
```
00 00 27 10    // DN 0: 10,000 quarters
00 00 4E 20    // DN 1: 20,000 ones
00 00 13 88    // DN 2: 5,000 fives
00 00 03 E8    // DN 3: 1,000 twenty-fives
00 00 01 F4    // DN 4: 500 hundreds
00 00 00 64    // DN 5: 100 two-fifty
00 00 00 0A    // DN 6: 10 thousands
3E 3E          // Terminator

Total value: 2,500 + 20,000 + 25,000 + 25,000 + 50,000 + 25,000 + 10,000
           = 157,500 units
```

## Use Cases

1. **Auditing**: Verify total currency in circulation
2. **Denomination Distribution**: Analyze token mix
3. **Consistency Check**: Compare counts across RAIDA
