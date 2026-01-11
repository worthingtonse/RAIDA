# 32-Byte Request Header

## Usage

Used with encryption types 0, 1, 2, and 3 (128-bit AES or no encryption).

## Byte Map

```
Offset: 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
        VR SP RI SH CG CM ID ID BF AP AP CP TR AI PI PC
        EN DN SN SN SN SN BL BL NO NO NO NO NO NO EC EC
```

## Field Reference

| Offset | Size | Field | Description | Value |
|--------|------|-------|-------------|-------|
| 0 | 1 | VR | Version | 0x01 |
| 1 | 1 | SP | Split ID | 0x00 (future) |
| 2 | 1 | RI | RAIDA ID | 0-24 |
| 3 | 1 | SH | Shard ID | 0x00 (future) |
| 4 | 1 | CG | Command Group | See commands |
| 5 | 1 | CM | Command Code | See commands |
| 6-7 | 2 | ID | Coin ID | 0x0001 (BE) |
| 8 | 1 | BF | Bitfield | 0x01 |
| 9-10 | 2 | AP | Application | 0x0000 |
| 11 | 1 | CP | Compression | 0x00 (future) |
| 12 | 1 | TR | Translation | 0x00 (future) |
| 13 | 1 | AI | AI Mode | 0x00 (future) |
| 14 | 1 | PI | Packet Index | 0x00 |
| 15 | 1 | PC | Packet Count | 0x01 |
| 16 | 1 | EN | Encryption Type | 0-3 |
| 17 | 1 | DN | Denomination | Key token DN |
| 18-21 | 4 | SN | Serial Number | Key token SN (BE) |
| 22-23 | 2 | BL | Body Length | Size in bytes (BE) |
| 24-29 | 6 | NO | Nonce | Random bytes |
| 30-31 | 2 | EC | Echo | Client tracking |

## Construction Example

```python
def build_header_32(
    raida_id: int,
    command_group: int,
    command_code: int,
    encryption_type: int,
    key_denomination: int,
    key_serial: int,
    body_length: int,
    nonce: bytes,
    echo: bytes
) -> bytes:
    header = bytearray(32)

    # Routing (0-7)
    header[0] = 0x01              # Version
    header[1] = 0x00              # Split ID
    header[2] = raida_id          # RAIDA ID
    header[3] = 0x00              # Shard ID
    header[4] = command_group     # Command Group
    header[5] = command_code      # Command Code
    header[6:8] = b'\x00\x01'     # Coin ID (big-endian)

    # Presentation (8-15)
    header[8] = 0x01              # Bitfield
    header[9:16] = b'\x00' * 7    # Reserved

    # Encryption ID (16-23)
    header[16] = encryption_type
    header[17] = key_denomination
    header[18:22] = key_serial.to_bytes(4, 'big')
    header[22:24] = body_length.to_bytes(2, 'big')

    # Nonce (24-31)
    header[24:30] = nonce[:6]
    header[30:32] = echo[:2]

    return bytes(header)
```
