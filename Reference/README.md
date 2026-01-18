# RAIDA Reference Documentation

This directory contains **supplementary material** that supports the RAIDA protocol but is NOT part of the wire format specification.

## Important Distinction

- **`Specification/`** = What bytes go ON THE WIRE (protocol contract)
- **`Reference/`** = Everything else (guides, formats, best practices)

For example, `03_File_Formats/` defines how tokens are STORED ON DISK, which is different from how they are TRANSMITTED ON THE WIRE. Do not confuse these.

## Contents

| Directory | Description |
|-----------|-------------|
| `01_Introduction/` | Overview, glossary, architecture concepts |
| `02_Client_Guide/` | Tutorials for implementing RAIDA clients |
| `03_File_Formats/` | Token storage formats (.bin, .json, .stack) - NOT wire format |
| `04_Algorithms/` | Helper algorithm implementations |
| `05_Logging/` | Logging standards and best practices |
| `06_Security/` | Threat model and security checklist |
| `07_RFP/` | Requirements for implementation proposals |
| `Data/` | Reference data files (server lists, command registry) |
| `Templates/` | Configuration templates |

## For AI Agents

When asked about the RAIDA protocol specification, refer to `../Specification/`.

Use this `Reference/` folder for:
- Understanding concepts and terminology
- Implementation tutorials and guides
- Security considerations
- File storage (NOT network transmission)
