# ECHO (Group 0, Code 0)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 0 |
| Command Code | 0 |
| Header Size | 32 bytes |
| Request Body | 18 bytes minimum |
| Response Body | None |
| Encryption | Optional |
| Transport | UDP preferred |

## Purpose

Tests connection and encryption between client and RAIDA server.
Used for health checks and debugging.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16-17 | 2 | Terminator | 0x3E 0x3E |

Total: 18 bytes minimum (can be larger for testing)

## Response

No body. Status code indicates result.

| Status | Meaning |
|--------|---------|
| 250 | Success |
| 37 | Invalid CRC in challenge |

## Algorithm

### Build Request
```pseudocode
function build_echo_request(raida_id: int) -> bytes:
    // Step 1: Build header
    header = build_header_32(
        raida_id = raida_id,
        command_group = 0,
        command_code = 0,
        encryption_type = 0,  // No encryption for basic echo
        body_length = 18
    )

    // Step 2: Build body
    challenge = generate_challenge()  // 16 bytes
    body = challenge + b'\x3E\x3E'

    return header + body
```

### Validate Response
```pseudocode
function validate_echo_response(response: bytes) -> bool:
    header = parse_response_header(response)
    return header.status == 250
```

## Example

### Request (Hex)
```
Header (32 bytes):
01 00 00 00 00 00 00 01 01 00 00 00 00 00 00 01
00 00 00 00 00 00 00 12 A1 B2 C3 D4 E5 F6 12 34

Body (18 bytes):
A1 B2 C3 D4 E5 F6 07 08 09 0A 0B 0C XX XX XX XX 3E 3E
                                    ^^^^^^^^^^^^ CRC32
```

### Response (32 bytes header only)
```
00 00 FA 00 00 00 12 34 00 00 00 00 00 00 00 00
[signature 16 bytes]
```
Status 0xFA = 250 = SUCCESS

## Test Vectors

### Test 1: Basic Echo
```json
{
  "input": {
    "raida_id": 0,
    "encryption_type": 0
  },
  "expected": {
    "status": 250
  }
}
```

### Test 2: Invalid CRC
```json
{
  "input": {
    "challenge": "000000000000000000000000FFFFFFFF"
  },
  "expected": {
    "status": 37
  }
}
```

## Common Mistakes

**DON'T** skip CRC validation even for unencrypted echo:
```python
# WRONG
challenge = os.urandom(16)  # No CRC!
```

**DO** always include valid CRC:
```python
# CORRECT
random_data = os.urandom(12)
crc = calculate_crc32(random_data)
challenge = random_data + crc
```
