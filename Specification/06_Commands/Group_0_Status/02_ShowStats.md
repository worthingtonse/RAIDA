# SHOW STATS (Group 0, Code 2)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 0 |
| Command Code | 2 |
| Header Size | 32 bytes |
| Request Body | 18 bytes |
| Response Body | Statistics data + terminator |
| Encryption | Optional |
| Transport | UDP preferred |

## Purpose

Retrieves operational statistics from RAIDA server.
Used for monitoring and diagnostics.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16-17 | 2 | Terminator | 0x3E 0x3E |

## Response

| Status | Body |
|--------|------|
| 250 (SUCCESS) | Statistics structure + terminator |

### Statistics Structure

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-3 | 4 | Uptime | Seconds since start (BE) |
| 4-7 | 4 | Requests | Total requests handled (BE) |
| 8-11 | 4 | Detects | Detect commands processed (BE) |
| 12-15 | 4 | Powns | POWN commands processed (BE) |
| 16-19 | 4 | Failures | Failed authentications (BE) |

## Algorithm

### Build Request
```pseudocode
function build_stats_request(raida_id: int) -> bytes:
    header = build_header_32(
        raida_id = raida_id,
        command_group = 0,
        command_code = 2,
        encryption_type = 0,
        body_length = 18
    )

    challenge = generate_challenge()
    body = challenge + b'\x3E\x3E'

    return header + body
```

### Parse Response
```pseudocode
function parse_stats_response(response: bytes) -> dict:
    header = parse_response_header(response)

    if header.status != 250:
        raise Error("Stats request failed")

    body = response[32:-2]
    return {
        'uptime': int.from_bytes(body[0:4], 'big'),
        'requests': int.from_bytes(body[4:8], 'big'),
        'detects': int.from_bytes(body[8:12], 'big'),
        'powns': int.from_bytes(body[12:16], 'big'),
        'failures': int.from_bytes(body[16:20], 'big')
    }
```

## Example

### Response Body
```
00 01 51 80    // Uptime: 86400 seconds (1 day)
00 0F 42 40    // Requests: 1,000,000
00 0A 00 00    // Detects: 655,360
00 05 00 00    // POWNs: 327,680
00 00 10 00    // Failures: 4,096
3E 3E          // Terminator
```

## Use Cases

1. **Health Monitoring**: Track server load and uptime
2. **Anomaly Detection**: Identify unusual failure rates
3. **Capacity Planning**: Monitor request volumes
