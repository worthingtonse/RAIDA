# Packet Structure

## Overview

All RAIDA protocol packets consist of three parts:

```
[Header][Body][Terminator]
```

## Request Packet

| Component | Size | Encrypted |
|-----------|------|-----------|
| Request Header | 32 or 48 bytes | No |
| Request Body | Variable | Yes (if EN > 0) |
| Terminator | 2 bytes | No |

## Response Packet

| Component | Size | Encrypted |
|-----------|------|-----------|
| Response Header | 32 bytes (always) | No |
| Response Body | Variable or none | No |
| Terminator | 2 bytes (if body present) | No |

## Header Size Selection

| Encryption Type | Header Size |
|-----------------|-------------|
| 0, 1, 2, 3 | 32 bytes |
| 4, 5, 7 | 48 bytes |

## Terminator Bytes

The terminator is always `0x3E 0x3E` and is NOT encrypted.

```
Position: [end of packet - 2]
Value: 0x3E 0x3E
```

## Byte Order

All multi-byte integers are **big-endian** (network byte order).

```python
# Example: Serial number 102205 (0x00018F3D)
serial_bytes = b'\x00\x01\x8F\x3D'  # Big-endian
```
