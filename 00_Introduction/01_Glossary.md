# Glossary

## Core Terms

| Term | Definition |
|------|------------|
| **AN** | Authenticity Number. 16-byte password for a token. |
| **PAN** | Proposed Authenticity Number. New AN to replace current. |
| **SN** | Serial Number. 4-byte unique identifier for a token. |
| **DN** | Denomination. 1-byte code indicating token value. |
| **RAIDA** | Redundant Array of Independent Detection Agents. |
| **POWN** | Password Own. Command to change AN to PAN. |

## Token States

| State | Description |
|-------|-------------|
| **Authentic** | Token passes on 13+ of 25 RAIDA |
| **Counterfeit** | Token fails on 13+ of 25 RAIDA |
| **Fractured** | Token has mixed results (1-12 failures) |
| **Limbo** | Transaction outcome unknown (no response) |

## Encryption Terms

| Term | Definition |
|------|------------|
| **Nonce** | Number used once. Prevents replay attacks. |
| **Challenge** | 16-byte field with 12 random bytes + 4-byte CRC32. |
| **Shared Secret** | Key known to both client and server. |

## Network Terms

| Term | Definition |
|------|------------|
| **Fan-out** | Sending requests to all 25 RAIDA in parallel. |
| **Consensus** | 13-of-25 RAIDA agreement required. |
| **Ticket** | Proof of authentication for healing. |
