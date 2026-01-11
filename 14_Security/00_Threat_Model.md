# Threat Model

## Overview

This document describes security threats and mitigations for RAIDA clients.

## Assets to Protect

| Asset | Value | Location |
|-------|-------|----------|
| Authenticity Numbers (AN) | Token ownership | Client storage |
| Tokens | Monetary value | Client storage |
| Network traffic | Transaction privacy | In transit |

## Threat Categories

### T1: AN Exposure

**Threat**: Attacker obtains Authenticity Numbers

**Attack Vectors**:
- Reading unencrypted storage
- Network sniffing (if unencrypted)
- Log file exposure
- Memory dumps

**Mitigations**:
- Always encrypt network traffic (encryption_type >= 1)
- Never log ANs (see Logging guide)
- Encrypt tokens at rest
- Use secure memory handling

### T2: RAIDA Impersonation

**Threat**: Attacker impersonates a RAIDA server

**Attack Vectors**:
- DNS spoofing
- Man-in-the-middle
- Rogue network access point

**Mitigations**:
- Validate response signatures (Challenge XOR AN)
- Require consensus from 13+ independent RAIDA
- Use DNS over HTTPS where available

### T3: Replay Attack

**Threat**: Attacker replays captured request

**Attack Vectors**:
- Capture valid POWN request
- Replay to transfer token

**Mitigations**:
- Random challenge per request
- RAIDA rejects duplicate requests
- ANs change after each POWN

### T4: Double Spend

**Threat**: Spend same token twice before RAIDA sync

**Attack Vectors**:
- Send to recipient A and B simultaneously
- Race condition between POWN requests

**Mitigations**:
- RAIDA atomically updates AN on POWN
- First valid POWN wins
- Second attempt fails authentication

## Protocol Security Properties

### Encryption Requirements

| Encryption Type | Security Level | Use Case |
|-----------------|---------------|----------|
| 0 (None) | None | Echo only |
| 1 (AES-128) | 128-bit | Standard operations |
| 4 (AES-256) | 256-bit | High-security |
| 5 (AES-256) | 256-bit | Two-token key |

### Consensus Security

With 25 RAIDA and 13 required for consensus:
- Attacker must compromise 13 servers to forge tokens
- Servers are geographically distributed
- Independent operators

### Challenge-Response Security

```
Request:  Client sends Challenge
Response: Server sends (Challenge XOR AN)

Only the server with correct AN can produce valid signature
```

## Client Security Checklist

### Storage Security

- [ ] Tokens encrypted at rest
- [ ] File permissions restrict access
- [ ] Backup encrypted separately
- [ ] No ANs in logs or temp files

### Network Security

- [ ] Encryption type >= 1 for all token operations
- [ ] Validate response signatures
- [ ] Use timeouts to prevent hanging
- [ ] Don't retry indefinitely

### Code Security

- [ ] Secure random for challenges
- [ ] Secure random for PANs
- [ ] Clear sensitive data from memory
- [ ] No secrets in error messages

## Incident Response

### If AN is Exposed

1. Immediately POWN token to new ANs
2. Send to all 25 RAIDA in parallel
3. Verify consensus achieved
4. Destroy old AN records

### If Token File is Stolen

1. Race to POWN before attacker
2. If attacker POWNs first, token is lost
3. Monitor for fractured state
4. Report to authorities if applicable
