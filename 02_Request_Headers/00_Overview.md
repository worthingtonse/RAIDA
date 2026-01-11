# Request Headers Overview

## Header Sizes

| Encryption Type | Header Size |
|-----------------|-------------|
| 0, 1, 2, 3 | 32 bytes |
| 4, 5, 7 | 48 bytes |

## Header Sections

Both header sizes contain four logical sections:

| Section | 32-byte Header | 48-byte Header |
|---------|----------------|----------------|
| Routing | Bytes 0-7 | Bytes 0-7 |
| Presentation | Bytes 8-15 | Bytes 8-15 |
| Encryption ID | Bytes 16-23 | Bytes 16-23 |
| Nonce | Bytes 24-31 | Bytes 24-47 |

## Routing Section (Bytes 0-7)

| Byte | Field | Description |
|------|-------|-------------|
| 0 | Version | Protocol version (0x01) |
| 1 | Split ID | Future use (0x00) |
| 2 | RAIDA ID | Target server (0-24) |
| 3 | Shard ID | Future use (0x00) |
| 4 | Command Group | High byte of command |
| 5 | Command Code | Low byte of command |
| 6-7 | Coin ID | Token type (0x0001 for WEST) |

## Presentation Section (Bytes 8-15)

| Byte | Field | Description |
|------|-------|-------------|
| 8 | Bitfield | Flags (0x01) |
| 9-10 | Application | App identifier (0x0000) |
| 11 | Compression | Future use (0x00) |
| 12 | Translation | Future use (0x00) |
| 13 | AI | Future use (0x00) |
| 14 | Packet Index | For multi-packet (0x00) |
| 15 | Packet Count | For multi-packet (0x01) |

## Encryption ID Section (Bytes 16-23)

| Byte | Field | Description |
|------|-------|-------------|
| 16 | Encryption Type | EN code (0-7) |
| 17 | Denomination | Key token denomination |
| 18-21 | Serial Number | Key token SN (big-endian) |
| 22-23 | Body Length | Encrypted body size |

## Quick Reference

See:
- `01_Header_32_Byte.md` for 32-byte header details
- `02_Header_48_Byte.md` for 48-byte header details
