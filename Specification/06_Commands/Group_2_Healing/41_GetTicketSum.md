# GET TICKET SUM (Group 2, Code 41)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 2 |
| Command Code | 41 |
| Header Size | 32 bytes |
| Request Body | 18 + (5 * N) + 16 bytes |
| Response Body | Tickets (16 * N) + terminator |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Optimized GetTicket using checksum instead of full ANs.
Reduces bandwidth when requesting tickets for many tokens.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16+ | 5*N | Tokens | N token identifiers |
| +16 | 16 | AN Checksum | MD5 of concatenated ANs |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

### Token Identifier (5 bytes each)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | Denomination |
| 1-4 | 4 | Serial Number (BE) |

## Response

| Status | Body |
|--------|------|
| 241 (ALL_PASS) | N × 16-byte tickets |
| 242 (ALL_FAIL) | None |
| 243 (MIXED) | Bitfield + valid tickets |

## Algorithm

### Build Request
```pseudocode
function build_get_ticket_sum_request(tokens: list, raida_id: int) -> bytes:
    checksum = calculate_an_checksum(tokens, raida_id)

    body_len = 16 + (5 * len(tokens)) + 16 + 2
    header = build_header_32(
        raida_id = raida_id,
        command_group = 2,
        command_code = 41,
        encryption_type = 1,
        key_dn = tokens[0].denomination,
        key_sn = tokens[0].serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    for token in tokens:
        body += bytes([token.denomination])
        body += token.serial_number.to_bytes(4, 'big')
    body += checksum

    padded = pad_body(body, encryption_type=1)
    key = tokens[0].an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

## Bandwidth Comparison

| Tokens | Standard GetTicket | GetTicketSum | Savings |
|--------|-------------------|--------------|---------|
| 10 | 210 bytes | 66 bytes | 69% |
| 50 | 1050 bytes | 266 bytes | 75% |

## Use Case

Use GetTicketSum when:
1. All tokens are known authentic on this RAIDA
2. Requesting tickets for batch healing
3. Bandwidth is constrained
