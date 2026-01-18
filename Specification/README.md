# RAIDA Protocol Specification

This directory contains the **authoritative specification** for the RAIDA protocol wire format. These documents define exactly what bytes are transmitted between client and server.

## For AI Agents

**The complete and authoritative protocol definition is contained within this directory.**

When implementing a RAIDA client or server, use ONLY the documents in this folder for the protocol specification. The `Reference/` folder contains supplementary material that is helpful but NOT part of the wire format contract.

## Contents

| Directory | Description |
|-----------|-------------|
| `01_Fundamentals/` | Packet structure, byte order, padding, terminators, networking (UDP/TCP) |
| `02_Request_Headers/` | 32-byte request header format |
| `03_Response_Headers/` | Response header format and status codes |
| `04_Cryptography/` | AES-128-CTR encryption, CRC32, challenge construction |
| `05_Denominations/` | Token denomination encoding (logarithmic codes) |
| `06_Commands/` | All protocol commands organized by group |

## Quick Start

1. Start with `01_Fundamentals/00_Packet_Structure.md` for the overall packet layout
2. Study `02_Request_Headers/01_Header_32_Byte.md` for the header format
3. Implement the Echo command (`06_Commands/Group_0_Status/00_Echo.md`) first
4. Add encryption using `04_Cryptography/`

## Key Principles

- All multi-byte integers are **big-endian**
- All packets end with **0x3E3E** terminator
- All encryption uses **AES-128-CTR** mode
- Denomination codes are **signed 8-bit** (two's complement)
- Consensus requires **13 of 25** RAIDA agreement
