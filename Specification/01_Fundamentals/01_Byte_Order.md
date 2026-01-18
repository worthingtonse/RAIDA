# Byte Order Specification

## Rule

All multi-byte integer fields in the RAIDA protocol MUST use **big-endian**
(network byte order).

## Definition

- Big-endian: Most Significant Byte (MSB) first
- Also called: Network byte order

## Examples

### Serial Number (4 bytes)

```
Decimal: 102205
Hex: 0x00018F3D

Byte order:
  Byte 0: 0x00 (MSB)
  Byte 1: 0x01
  Byte 2: 0x8F
  Byte 3: 0x3D (LSB)

Wire format: 00 01 8F 3D
```

### Body Length (2 bytes)

```
Decimal: 1234
Hex: 0x04D2

Byte order:
  Byte 0: 0x04 (MSB)
  Byte 1: 0xD2 (LSB)

Wire format: 04 D2
```

## Implementation

### Python
```python
value = 102205
bytes_be = value.to_bytes(4, byteorder='big')
# Result: b'\x00\x01\x8f='
```

### Go
```go
import "encoding/binary"
buf := make([]byte, 4)
binary.BigEndian.PutUint32(buf, 102205)
```

### C
```c
uint32_t value = 102205;
uint32_t be_value = htonl(value);
```

## Common Mistake

**DON'T** use little-endian:
```python
# WRONG
bytes_le = value.to_bytes(4, byteorder='little')
# Result: b'=\x8f\x01\x00' - INCORRECT
```
