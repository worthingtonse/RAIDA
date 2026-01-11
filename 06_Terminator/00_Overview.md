# Terminator Bytes

## Overview

All RAIDA packets end with a 2-byte terminator sequence.

## Terminator Value

```
Hex: 0x3E 0x3E
ASCII: ">>"
Decimal: 62 62
```

## Packet Structure

```
[Header 32/48 bytes][Body (encrypted)][Terminator 0x3E3E]
```

## Purpose

1. **Packet Boundary Detection**: Identifies end of packet in stream
2. **Corruption Detection**: Missing terminator indicates truncation
3. **Protocol Validation**: Wrong terminator indicates parsing error

## Position in Packet

The terminator is ALWAYS:
- The last 2 bytes of the packet
- AFTER the encrypted body
- NOT included in body_size field

## Implementation

### Appending Terminator

```python
def build_packet(header: bytes, encrypted_body: bytes) -> bytes:
    return header + encrypted_body + b'\x3E\x3E'
```

### Validating Terminator

```python
def validate_packet(packet: bytes) -> bool:
    if len(packet) < 34:  # Min: 32 header + 2 terminator
        return False
    return packet[-2:] == b'\x3E\x3E'
```

### Stripping Terminator

```python
def parse_response(packet: bytes) -> tuple:
    if not validate_packet(packet):
        raise ValueError("Invalid terminator")

    header = packet[:32]
    body = packet[32:-2]  # Exclude terminator
    return header, body
```

## Common Mistakes

**DON'T** include terminator in body_size:
```python
# WRONG
body_size = len(encrypted_body) + 2  # Don't count terminator!
```

**DO** only count body in body_size:
```python
# CORRECT
body_size = len(encrypted_body)  # Terminator not included
```

**DON'T** encrypt the terminator:
```python
# WRONG
encrypted = aes_encrypt(body + b'\x3E\x3E')
```

**DO** append terminator after encryption:
```python
# CORRECT
encrypted = aes_encrypt(body)
packet = header + encrypted + b'\x3E\x3E'
```

**DON'T** forget terminator on responses:
```python
# WRONG - Server response
response = header  # No terminator!
```

**DO** include terminator on all packets:
```python
# CORRECT
response = header + b'\x3E\x3E'  # Even if no body
```
