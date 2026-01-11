# VERSION (Group 0, Code 1)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 0 |
| Command Code | 1 |
| Header Size | 32 bytes |
| Request Body | 18 bytes |
| Response Body | Version string + terminator |
| Encryption | Optional |
| Transport | UDP preferred |

## Purpose

Retrieves the RAIDA server software version.
Used for compatibility checking and diagnostics.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16-17 | 2 | Terminator | 0x3E 0x3E |

## Response

| Status | Body |
|--------|------|
| 250 (SUCCESS) | Version string + terminator |

### Response Body Format

```
[Version String (variable length)][0x3E 0x3E]
```

Version string is ASCII text, typically in format: `MAJOR.MINOR.PATCH`

## Algorithm

### Build Request
```pseudocode
function build_version_request(raida_id: int) -> bytes:
    header = build_header_32(
        raida_id = raida_id,
        command_group = 0,
        command_code = 1,
        encryption_type = 0,
        body_length = 18
    )

    challenge = generate_challenge()
    body = challenge + b'\x3E\x3E'

    return header + body
```

### Parse Response
```pseudocode
function parse_version_response(response: bytes) -> str:
    header = parse_response_header(response)

    if header.status != 250:
        raise Error("Version request failed")

    // Body is ASCII string before terminator
    body = response[32:-2]
    return body.decode('ascii')
```

## Example

### Request
```
Header (32 bytes): standard header with CG=0, CC=1
Body: [16B challenge] 3E 3E
```

### Response
```
Header: ... status=0xFA (250) ...
Body: 33 2E 31 2E 34 3E 3E
      "3.1.4" + terminator
```

## Use Cases

1. **Compatibility Check**: Verify server supports required features
2. **Diagnostics**: Log server versions for debugging
3. **Monitoring**: Track server software across network

## Common Mistakes

**DON'T** assume fixed version string length:
```python
# WRONG
version = response[32:37]  # Assumes 5 chars
```

**DO** parse until terminator:
```python
# CORRECT
body = response[32:-2]  # Everything except terminator
version = body.decode('ascii')
```
