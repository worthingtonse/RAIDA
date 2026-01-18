# RFP Requirements Specification

## Overview

This document specifies requirements for RAIDA client implementations.
Use this as a checklist for Request for Proposal (RFP) responses.

## Mandatory Requirements (MUST)

### M1: Protocol Compliance

| ID | Requirement |
|----|-------------|
| M1.1 | MUST implement 32-byte request header format exactly |
| M1.2 | MUST implement encryption types 0, 1, and 2 |
| M1.3 | MUST use big-endian byte order for all multi-byte integers |
| M1.4 | MUST append 0x3E3E terminator to all packets |
| M1.5 | MUST include 16-byte challenge at start of body for all encryption types |

### M2: Cryptography

| ID | Requirement |
|----|-------------|
| M2.1 | MUST implement AES-CTR mode encryption |
| M2.2 | MUST use zlib-compatible CRC32 for challenge validation |
| M2.3 | MUST validate response signatures (Challenge XOR AN) |
| M2.4 | MUST use cryptographically secure random for challenges and PANs |
| M2.5 | MUST use 8-byte nonce from header bytes 24-31 for AES-CTR |

### M3: Networking

| ID | Requirement |
|----|-------------|
| M3.1 | MUST support UDP transport on ports 18000-18024 |
| M3.2 | MUST implement configurable timeouts |
| M3.3 | MUST support parallel requests to all 25 RAIDA |
| M3.4 | MUST handle partial responses and network errors gracefully |

### M4: Commands

| ID | Requirement |
|----|-------------|
| M4.1 | MUST implement Echo command (Group 0, Code 0) |
| M4.2 | MUST implement Detect command (Group 1, Code 10) |
| M4.3 | MUST implement POWN command (Group 1, Code 20) |
| M4.4 | MUST parse status codes correctly (241, 242, 243) |
| M4.5 | MUST parse bitfield responses for mixed results |

### M5: Consensus

| ID | Requirement |
|----|-------------|
| M5.1 | MUST require 13+ RAIDA for consensus |
| M5.2 | MUST track per-RAIDA results separately |
| M5.3 | MUST handle fractured tokens (< 13 consensus) |

## Recommended Requirements (SHOULD)

### R1: Performance

| ID | Requirement |
|----|-------------|
| R1.1 | SHOULD complete 25-RAIDA operation in < 5 seconds |
| R1.2 | SHOULD support batch operations (multiple tokens per request) |
| R1.3 | SHOULD implement connection reuse for TCP |

### R2: Reliability

| ID | Requirement |
|----|-------------|
| R2.1 | SHOULD implement retry with exponential backoff |
| R2.2 | SHOULD implement adaptive timeouts based on history |
| R2.3 | SHOULD support TCP transport for large payloads |

### R3: Features

| ID | Requirement |
|----|-------------|
| R3.1 | SHOULD implement healing (GetTicket, Fix) |
| R3.2 | SHOULD implement denomination change (Break, Join) |
| R3.3 | SHOULD implement locker operations (Put, Peek, Remove) |

### R4: Security

| ID | Requirement |
|----|-------------|
| R4.1 | SHOULD encrypt tokens at rest |
| R4.2 | SHOULD never log sensitive data (AN, PAN, keys) |
| R4.3 | SHOULD validate all server responses |

## Optional Requirements (MAY)

### O1: User Interface

| ID | Requirement |
|----|-------------|
| O1.1 | MAY provide CLI interface |
| O1.2 | MAY provide GUI interface |
| O1.3 | MAY provide REST API |

### O2: Integration

| ID | Requirement |
|----|-------------|
| O2.1 | MAY support wallet file formats (binary, JSON) |
| O2.2 | MAY support import/export of tokens |
| O2.3 | MAY integrate with external key management |

## Compliance Testing

### Test Categories

| Category | Description |
|----------|-------------|
| Unit | Individual function correctness |
| Integration | End-to-end command execution |
| Network | Real RAIDA server communication |
| Stress | High-volume parallel operations |

### Minimum Test Coverage

```
- Echo to all 25 RAIDA
- Detect single token (pass and fail cases)
- Detect batch (mixed results)
- POWN single token
- POWN with consensus verification
- Timeout handling
- Retry behavior
```

## Deliverables

### Documentation

- [ ] Architecture overview
- [ ] API reference
- [ ] Configuration guide
- [ ] Security considerations

### Source Code

- [ ] Core protocol implementation
- [ ] Unit tests
- [ ] Integration tests
- [ ] Example applications

### Artifacts

- [ ] Build instructions
- [ ] Deployment guide
- [ ] Test reports
