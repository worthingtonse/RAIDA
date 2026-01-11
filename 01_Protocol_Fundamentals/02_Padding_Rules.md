# Padding Rules

## Purpose

Request bodies MUST be padded to align with encryption block sizes.

## Block Sizes by Encryption Type

| Encryption Type | Block Size | Padding Required |
|-----------------|------------|------------------|
| 0 (None) | N/A | No padding |
| 1, 2, 3 (AES-128) | 16 bytes | Yes |
| 4, 5, 7 (AES-256) | 32 bytes | Yes |

## Padding Specification

- Padding byte value: `0x00`
- Padding location: After body content, BEFORE terminator
- Terminator is NOT included in padding calculation

## Algorithm

```python
def pad_body(body: bytes, encryption_type: int) -> bytes:
    """
    Pad body to required block alignment.

    Args:
        body: Unpadded body content (excludes terminator)
        encryption_type: EN value from header

    Returns:
        Padded body (terminator still needs to be appended)
    """
    if encryption_type == 0:
        return body  # No padding

    block_size = 32 if encryption_type in [4, 5, 7] else 16

    remainder = len(body) % block_size
    if remainder == 0:
        return body  # Already aligned

    padding_needed = block_size - remainder
    return body + bytes(padding_needed)  # Append 0x00 bytes
```

## Example

```
Body content:     23 bytes
Encryption type:  1 (AES-128, 16-byte blocks)
Padding needed:   16 - (23 % 16) = 16 - 7 = 9 bytes
Padded body:      23 + 9 = 32 bytes

Final packet:
[32-byte header][32-byte encrypted padded body][0x3E 0x3E]
```

## Common Mistake

**DON'T** include terminator in padding calculation:
```python
# WRONG - terminator is outside encryption
padded = pad_body(body + b'\x3E\x3E', en_type)
```

**DO** pad body first, then append terminator:
```python
# CORRECT
padded = pad_body(body, en_type)
packet = header + encrypt(padded) + b'\x3E\x3E'
```
