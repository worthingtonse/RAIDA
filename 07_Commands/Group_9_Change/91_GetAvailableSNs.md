# GET AVAILABLE SNs (Group 9, Code 91)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 9 |
| Command Code | 91 |
| Header Size | 32 bytes |
| Request Body | 19 bytes |
| Response Body | SN list + terminator |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Retrieves available serial numbers for a denomination.
Required before Break/Join to know which SNs can be used.

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16 | 1 | Target DN | Denomination to query |
| 17-18 | 2 | Terminator | 0x3E 0x3E |

## Response

| Status | Body |
|--------|------|
| 250 (SUCCESS) | Available SN list |
| 160 (CHANGE_LIMIT) | Rate limit exceeded |

### Success Response Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-1 | 2 | Count | Number of SNs (BE) |
| 2+ | 4*N | SNs | N serial numbers (BE) |

## Algorithm

### Build Request
```pseudocode
function build_get_sns_request(
    raida_id: int,
    target_denomination: int,
    key_token: Token
) -> bytes:
    body_len = 16 + 1 + 2

    header = build_header_32(
        raida_id = raida_id,
        command_group = 9,
        command_code = 91,
        encryption_type = 1,
        key_dn = key_token.denomination,
        key_sn = key_token.serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    body += bytes([target_denomination])

    padded = pad_body(body, encryption_type=1)
    key = key_token.an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

### Parse Response
```pseudocode
function parse_get_sns_response(response: bytes) -> list:
    header = parse_response_header(response)

    if header.status == 160:
        raise RateLimitError("Change limit exceeded")

    if header.status != 250:
        raise Error("GetAvailableSNs failed")

    body = response[32:-2]
    count = int.from_bytes(body[0:2], 'big')

    sns = []
    for i in range(count):
        offset = 2 + (i * 4)
        sn = int.from_bytes(body[offset:offset+4], 'big')
        sns.append(sn)

    return sns
```

## Use in Break/Join

```pseudocode
function prepare_for_break(token: Token, target_dn: int) -> list:
    // Get available SNs from all 25 RAIDA
    all_sns = {}

    for raida_id in range(25):
        sns = get_available_sns(raida_id, target_dn, token)
        for sn in sns:
            if sn not in all_sns:
                all_sns[sn] = set()
            all_sns[sn].add(raida_id)

    // Find SNs available on all 25 RAIDA
    consensus_sns = [sn for sn, raidas in all_sns.items()
                     if len(raidas) == 25]

    return consensus_sns
```

## Rate Limiting

RAIDA may limit change requests to prevent abuse:
- Status 160 indicates rate limit
- Wait and retry after delay
- Consider using different RAIDA

## Common Mistakes

**DON'T** assume SNs are available on all RAIDA:
```python
# WRONG - SN may not be available on all RAIDA
sns = get_available_sns(raida_0, target_dn)
use_sn = sns[0]  # May not work on RAIDA 1-24!
```

**DO** find consensus SNs:
```python
# CORRECT
all_results = [get_available_sns(r, dn) for r in range(25)]
consensus_sns = find_common_sns(all_results)
```
