# Response Header

## Size

Response headers are always 32 bytes, regardless of encryption type.

## Byte Map

```
Offset: 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
        RI SH SS CG UD UD EC EC RE SZ SZ SZ EX EX EX EX
        HS HS HS HS HS HS HS HS HS HS HS HS HS HS HS HS
```

## Field Reference

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 1 | RI | Responding RAIDA ID (0-24) |
| 1 | 1 | SH | Shard ID (0x00) |
| 2 | 1 | SS | Status Code |
| 3 | 1 | CG | Command Group (echo) |
| 4-5 | 2 | UD | UDP Frame Count |
| 6-7 | 2 | EC | Client Echo (from request) |
| 8 | 1 | RE | Reserved |
| 9-11 | 3 | SZ | Body Size (bytes) |
| 12-15 | 4 | EX | Execution Time (nanoseconds) |
| 16-31 | 16 | HS | Signature (XOR challenge) |

## Signature Field

The signature (bytes 16-31) is computed as:

```
Signature = Challenge XOR Encryption_AN
```

Where:
- Challenge = 16-byte challenge from request
- Encryption_AN = The AN used for encryption

## Parsing Example

```python
def parse_response_header(data: bytes) -> dict:
    if len(data) < 32:
        raise ValueError("Response too short")

    return {
        'raida_id': data[0],
        'shard_id': data[1],
        'status': data[2],
        'command_group': data[3],
        'udp_frame': int.from_bytes(data[4:6], 'big'),
        'echo': data[6:8],
        'body_size': int.from_bytes(data[9:12], 'big'),
        'exec_time_ns': int.from_bytes(data[12:16], 'big'),
        'signature': data[16:32]
    }
```

## Signature Validation

```python
def validate_signature(
    response_signature: bytes,
    sent_challenge: bytes,
    encryption_an: bytes
) -> bool:
    expected = bytes(a ^ b for a, b in zip(sent_challenge, encryption_an))
    return response_signature == expected
```
