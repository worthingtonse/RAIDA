# RAIDA Protocol Documentation

The Redundant Array of Independent Detection Agents (RAIDA) protocol enables secure, distributed authentication of digital tokens across 25 independent servers.

## Documentation Structure

This documentation is organized into three main directories:

```
RAIDA/
├── Specification/    <- Protocol wire format (THE CONTRACT)
├── Reference/        <- Supplementary guides and materials
└── _Meta/            <- Working files and collaboration notes
```

### Specification/ (Protocol Contract)

**The authoritative source for the RAIDA protocol wire format.**

Use this directory exclusively when implementing client-server communication. It defines exactly what bytes are transmitted on the network.

| Directory | Contents |
|-----------|----------|
| `01_Fundamentals/` | Packet structure, byte order, networking |
| `02_Request_Headers/` | 32-byte request header format |
| `03_Response_Headers/` | Response format and status codes |
| `04_Cryptography/` | AES-128-CTR, CRC32, challenge construction |
| `05_Denominations/` | Token denomination encoding |
| `06_Commands/` | All protocol commands by group |

### Reference/ (Supplementary)

Helpful guides and materials that are NOT part of the protocol specification.

| Directory | Contents |
|-----------|----------|
| `01_Introduction/` | Overview, glossary, architecture |
| `02_Client_Guide/` | Implementation tutorials |
| `03_File_Formats/` | Token disk storage formats |
| `04_Algorithms/` | Helper algorithm implementations |
| `05_Logging/` | Logging best practices |
| `06_Security/` | Threat model and checklist |
| `07_RFP/` | Requirements specification |

### _Meta/

Working files from documentation development. Not part of the final documentation.

## For AI Agents

**Simple instruction:** The complete protocol specification is in `/Specification`. For implementation guidance, see `/Reference`.

## Quick Start

1. Read `Specification/01_Fundamentals/00_Packet_Structure.md`
2. Study `Specification/02_Request_Headers/01_Header_32_Byte.md`
3. Implement Echo command: `Specification/06_Commands/Group_0_Status/00_Echo.md`

## Key Protocol Facts

- All multi-byte integers: **Big-endian**
- Packet terminator: **0x3E3E**
- Encryption: **AES-128-CTR**
- Consensus threshold: **13 of 25 RAIDA**
- Denomination codes: **Signed 8-bit** (two's complement)

## Related Resources

- [RAIDA Server (C)](https://github.com/worthingtonse/RAIDAX/tree/main/code) - Reference server implementation
