# FIX (Group 2, Code 80)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 2 |
| Command Code | 80 |
| Header Size | 32 bytes |
| Request Body | Variable |
| Response Body | None on success |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Repairs a fractured token by providing tickets from trusted RAIDA.
The target RAIDA validates tickets and updates its AN record.

## Concept

```
Token fractured: RAIDA 0-12 have AN, RAIDA 13-24 don't

Step 1: Get tickets from RAIDA 0-12 (they know the AN)
Step 2: Send Fix to RAIDA 13-24 with those tickets
Step 3: RAIDA 13-24 validate tickets and store AN
```

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16 | 1 | Denomination | Token denomination |
| 17-20 | 4 | Serial Number | Token SN (BE) |
| 21-36 | 16 | AN | The correct AN to store |
| 37 | 1 | Ticket Count | Number of tickets (13-25) |
| 38+ | 17*N | Tickets | N ticket records |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

### Ticket Record (17 bytes each)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | Source RAIDA ID |
| 1-16 | 16 | Ticket |

## Response

| Status | Meaning |
|--------|---------|
| 250 (SUCCESS) | AN updated successfully |
| 242 (FAIL) | Not enough valid tickets |

## Algorithm

### Build Request
```pseudocode
function build_fix_request(
    raida_id: int,
    token: Token,
    tickets: list  // List of (source_raida, ticket) tuples
) -> bytes:
    ticket_count = len(tickets)
    body_len = 16 + 1 + 4 + 16 + 1 + (17 * ticket_count) + 2

    header = build_header_32(
        raida_id = raida_id,
        command_group = 2,
        command_code = 80,
        encryption_type = 1,
        key_dn = token.denomination,
        key_sn = token.serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    body += bytes([token.denomination])
    body += token.serial_number.to_bytes(4, 'big')
    body += token.an[raida_id]  // AN we want RAIDA to store
    body += bytes([ticket_count])

    for source_raida, ticket in tickets:
        body += bytes([source_raida])
        body += ticket

    padded = pad_body(body, encryption_type=1)
    // Note: Use AN from a working RAIDA for encryption key
    key = get_working_an(token)
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

### Complete Healing Workflow
```pseudocode
function heal_token(token: Token) -> bool:
    // Step 1: Detect to find which RAIDA are working
    detect_results = detect_all_raida(token)

    working = [i for i, r in enumerate(detect_results) if r]
    broken = [i for i, r in enumerate(detect_results) if not r]

    if len(working) < 13:
        return False  // Can't heal - not enough working RAIDA

    if len(broken) == 0:
        return True   // Already healthy

    // Step 2: Get tickets from working RAIDA
    tickets = []
    for raida_id in working[:13]:  // Need minimum 13
        ticket = get_ticket(raida_id, token)
        if ticket:
            tickets.append((raida_id, ticket))

    if len(tickets) < 13:
        return False  // Couldn't get enough tickets

    // Step 3: Fix each broken RAIDA
    for raida_id in broken:
        fix(raida_id, token, tickets)

    // Step 4: Verify healing
    new_results = detect_all_raida(token)
    return sum(new_results) >= 13
```

## Ticket Requirements

- Minimum 13 tickets required
- Tickets must be from different RAIDA
- Tickets must not be expired
- All tickets must be for the same token

## Example

### Fix Request for RAIDA 15
```
Body:
[16B challenge]
01                      // Denomination
00 01 8F 3D             // SN = 102205
[16B AN to store]       // The correct AN
0D                      // 13 tickets

00 [16B ticket from R0]
01 [16B ticket from R1]
02 [16B ticket from R2]
...
0C [16B ticket from R12]

3E 3E
```

## Common Mistakes

**DON'T** send tickets from broken RAIDA:
```python
# WRONG - RAIDA 15 doesn't know the AN
ticket = get_ticket(15, token)  # Will fail
tickets.append((15, ticket))
```

**DO** only use tickets from working RAIDA:
```python
# CORRECT
for raida_id in working_raida:
    ticket = get_ticket(raida_id, token)
    tickets.append((raida_id, ticket))
```

**DON'T** send fewer than 13 tickets:
```python
# WRONG - Will be rejected
fix(raida_id, token, tickets[:10])
```

**DO** always send at least 13 tickets:
```python
# CORRECT
if len(tickets) >= 13:
    fix(raida_id, token, tickets[:13])
```

**DON'T** delay between getting tickets and fixing:
```python
# RISKY - Tickets may expire
tickets = get_all_tickets(token)
time.sleep(600)  # 10 minutes
fix(broken_raida, token, tickets)  # May fail!
```

**DO** use tickets immediately:
```python
# CORRECT
tickets = get_all_tickets(token)
for raida_id in broken_raida:
    fix(raida_id, token, tickets)  # Use right away
```
