# VALIDATE TICKET (Group 2, Code 50)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 2 |
| Command Code | 50 |
| Header Size | 32 bytes |
| Request Body | Variable |
| Response Body | Validation result + terminator |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Validates a healing ticket without applying it.
Used to verify ticket authenticity before Fix operation.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16 | 1 | Source RAIDA | RAIDA ID that issued ticket |
| 17 | 1 | Denomination | Token denomination |
| 18-21 | 4 | Serial Number | Token SN (BE) |
| 22-37 | 16 | Ticket | Ticket to validate |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

## Response

| Status | Meaning |
|--------|---------|
| 250 (SUCCESS) | Ticket is valid |
| 242 (FAIL) | Ticket is invalid or expired |

## Algorithm

### Build Request
```pseudocode
function build_validate_ticket_request(
    raida_id: int,
    source_raida: int,
    token: Token,
    ticket: bytes
) -> bytes:
    body_len = 16 + 1 + 1 + 4 + 16 + 2
    header = build_header_32(
        raida_id = raida_id,
        command_group = 2,
        command_code = 50,
        encryption_type = 1,
        key_dn = token.denomination,
        key_sn = token.serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    body += bytes([source_raida])
    body += bytes([token.denomination])
    body += token.serial_number.to_bytes(4, 'big')
    body += ticket

    padded = pad_body(body, encryption_type=1)
    key = token.an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

## Ticket Expiration

Tickets have a limited lifetime:
- Typically valid for 5-10 minutes
- Exact expiration is implementation-specific
- Always use tickets promptly after obtaining

## Use Cases

1. **Pre-validation**: Check tickets before Fix to avoid wasted requests
2. **Debugging**: Verify ticket generation is working
3. **Auditing**: Confirm ticket source and validity

## Common Mistakes

**DON'T** validate tickets you just received:
```python
# UNNECESSARY - Fresh tickets are valid
ticket = get_ticket(raida_5, token)
validate_ticket(raida_10, raida_5, token, ticket)  # Redundant
fix(raida_10, token, tickets)
```

**DO** validate only when tickets are old or from storage:
```python
# CORRECT - Validate cached/stored tickets
if ticket_age > 60:  # Over 1 minute old
    if validate_ticket(raida_id, source, token, ticket):
        fix(raida_id, token, tickets)
    else:
        # Get fresh tickets
        tickets = get_fresh_tickets(token)
```
