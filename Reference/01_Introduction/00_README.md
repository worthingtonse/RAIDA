# RAIDA Protocol Documentation

## Overview

The RAIDA (Redundant Array of Independent Detection Agents) protocol enables
secure, distributed authentication of digital tokens across 25 independent
servers.

## Documentation Structure

The documentation is organized into two main sections:

### Specification/ (Protocol Wire Format)
| Section | Description |
|---------|-------------|
| 01_Fundamentals | Packet structure, byte order, terminators, networking |
| 02_Request_Headers | 32-byte request headers |
| 03_Response_Headers | Response format and status codes |
| 04_Cryptography | Encryption, CRC32, challenge construction |
| 05_Denominations | Token denomination codes |
| 06_Commands | All protocol commands by group |

### Reference/ (Supplementary Material)
| Section | Description |
|---------|-------------|
| 01_Introduction | Overview, glossary, architecture |
| 02_Client_Guide | Client implementation guidance |
| 03_File_Formats | Token file formats (disk storage) |
| 04_Algorithms | Helper algorithms with pseudocode |
| 05_Logging | Logging standards |
| 06_Security | Security model and threats |
| 07_RFP | RFP compliance and acceptance criteria |

## Quick Start

1. Read `../../Specification/01_Fundamentals/00_Packet_Structure.md`
2. Review `../../Specification/02_Request_Headers/01_Header_32_Byte.md`
3. Implement `../../Specification/06_Commands/Group_0_Status/00_Echo.md` first

## Normative Language

This documentation uses RFC 2119 keywords:
- **MUST** / **MUST NOT**: Absolute requirement
- **SHOULD** / **SHOULD NOT**: Recommended
- **MAY**: Optional

## Version

Protocol Version: 1.0
Documentation Version: 1.0.0
