# Request Headers Overview

## Header Size

All encryption types use a 32-byte header.

## Encryption Types

| Type | Name | Status | Key Source |
|------|------|--------|------------|
| 0 | Unencrypted | Supported | None |
| 1 | AES-128 | Supported | Token AN |
| 2 | AES-128 Locker | Supported | Locker AN |
| 3 | Reserved | Unsupported | N/A |

**Note:** Type 3 is reserved for future use. Clients MUST NOT send requests with encryption type 3. Servers WILL reject such requests with `ERROR_INVALID_ENCRYPTION`.

## Header Sections

The header contains four logical sections:

| Section | Bytes | Description |
|---------|-------|-------------|
| Routing | 0-7 | Target server and command |
| Presentation | 8-15 | Flags and packet info |
| Encryption ID | 16-23 | Encryption config and body length |
| Nonce/Echo | 24-31 | Nonce bytes and echo |

## Routing Section (Bytes 0-7)

| Byte | Field | Description |
|------|-------|-------------|
| 0 | Version | Protocol version (0x01) |
| 1 | Split ID | Reserved (0x00) |
| 2 | RAIDA ID | Target server (0-24) |
| 3 | Shard ID | Reserved (0x00) |
| 4 | Command Group | Command group code |
| 5 | Command Code | Command code |
| 6-7 | Coin ID | Token type (0x0001 big-endian) |

## Presentation Section (Bytes 8-15)

| Byte | Field | Description |
|------|-------|-------------|
| 8 | Bitfield | Flags (0x01) |
| 9-10 | Application Port | Reserved (0x0000) |
| 11 | Compression | Reserved (0x00) |
| 12 | Translation | Reserved (0x00) |
| 13 | AI | Reserved (0x00) |
| 14 | Packet Index | For multi-packet UDP (0x00) |
| 15 | Packet Count | For multi-packet UDP (0x01) |

## Encryption ID Section (Bytes 16-23)

This section varies by encryption type:

| Byte | Type 0 | Type 1/2 |
|------|--------|----------|
| 16 | EN = 0x00 | EN = 0x01 or 0x02 |
| 17 | Ignored | Key Denomination |
| 18-21 | Ignored | Key Serial Number (big-endian) |
| 22-23 | Body Length | Body Length |

## Nonce/Echo Section (Bytes 24-31)

This section also varies by encryption type:

| Byte | Type 0 | Type 1/2 |
|------|--------|----------|
| 24-29 | Ignored | Nonce bytes 0-5 |
| 30-31 | Echo | Nonce bytes 6-7 AND Echo |

**Important:** For Types 1 and 2, bytes 30-31 serve a dual purpose:
- They are the last 2 bytes of the 8-byte nonce (used for AES-CTR)
- They are also read as echo bytes (returned in response)

## Quick Reference

See:
- `01_Header_32_Byte.md` for complete header details with Python examples
