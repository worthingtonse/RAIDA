# BREAK (Group 9, Code 92)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 9 |
| Command Code | 92 |
| Header Size | 32 bytes |
| Request Body | Variable |
| Response Body | New tokens + terminator |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Breaks a larger denomination token into smaller ones.
Example: Break one 25-unit into five 5-units.

## Denomination Values

| Code | Value | Break Into |
|------|-------|------------|
| 5 | 250 | 2×100 + 1×25 + 1×5 × 5 |
| 4 | 100 | 4×25 |
| 3 | 25 | 5×5 |
| 2 | 5 | 5×1 |
| 1 | 1 | 4×0.25 |

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16 | 1 | Source DN | Denomination to break |
| 17-20 | 4 | Source SN | Serial number (BE) |
| 21-36 | 16 | Source AN | Current AN |
| 37 | 1 | Target Count | Number of output tokens |
| 38+ | 21*N | Targets | Target token specs |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

### Target Token Spec (21 bytes each)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | Denomination |
| 1-4 | 4 | Serial Number (from GetAvailableSNs) |
| 5-20 | 16 | PAN (new AN for this token) |

## Response

| Status | Body |
|--------|------|
| 250 (SUCCESS) | Confirmation |
| 160 (CHANGE_LIMIT) | Rate limit exceeded |
| 242 (FAIL) | Invalid source or targets |

## Algorithm

### Build Request
```pseudocode
function build_break_request(
    raida_id: int,
    source_token: Token,
    target_specs: list  // [(dn, sn, pan), ...]
) -> bytes:
    target_count = len(target_specs)
    body_len = 16 + 1 + 4 + 16 + 1 + (21 * target_count) + 2

    header = build_header_32(
        raida_id = raida_id,
        command_group = 9,
        command_code = 92,
        encryption_type = 1,
        key_dn = source_token.denomination,
        key_sn = source_token.serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    body += bytes([source_token.denomination])
    body += source_token.serial_number.to_bytes(4, 'big')
    body += source_token.an[raida_id]
    body += bytes([target_count])

    for dn, sn, pan in target_specs:
        body += bytes([dn])
        body += sn.to_bytes(4, 'big')
        body += pan[raida_id]

    padded = pad_body(body, encryption_type=1)
    key = source_token.an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

### Complete Break Workflow
```pseudocode
function break_token(source: Token, target_dn: int) -> list:
    // Step 1: Calculate how many target tokens
    source_value = DENOMINATION_VALUES[source.denomination]
    target_value = DENOMINATION_VALUES[target_dn]
    target_count = source_value // target_value

    // Step 2: Get available SNs for target denomination
    available_sns = {}
    for raida_id in range(25):
        sns = get_available_sns(raida_id, target_dn, source)
        available_sns[raida_id] = sns

    // Find SNs available on all RAIDA
    common_sns = find_common_sns(available_sns, target_count)
    if len(common_sns) < target_count:
        raise Error("Not enough available serial numbers")

    // Step 3: Generate PANs for new tokens
    target_specs = []
    for i in range(target_count):
        sn = common_sns[i]
        pan = [os.urandom(16) for _ in range(25)]
        target_specs.append((target_dn, sn, pan))

    // Step 4: Execute break on all RAIDA
    results = []
    for raida_id in range(25):
        result = break_request(raida_id, source, target_specs)
        results.append(result)

    // Step 5: Check consensus and build new tokens
    if sum(results) >= 13:
        new_tokens = []
        for dn, sn, pan in target_specs:
            new_tokens.append(Token(dn, sn, pan))
        return new_tokens
    else:
        raise Error("Break failed - no consensus")
```

## Value Conservation

Break MUST preserve total value:

| Source | Targets | Total |
|--------|---------|-------|
| 1×25 | 5×5 | 25 |
| 1×100 | 4×25 | 100 |
| 1×5 | 5×1 | 5 |

## Common Mistakes

**DON'T** reuse serial numbers:
```python
# WRONG - SN may already be used
break(source, [(1, 12345, pan)])  # Random SN!
```

**DO** use GetAvailableSNs first:
```python
# CORRECT
available = get_available_sns(raida_id, target_dn)
break(source, [(target_dn, available[0], pan)])
```

**DON'T** break without consensus on SNs:
```python
# WRONG - SNs may differ across RAIDA
sn = get_available_sns(raida_0, dn)[0]
for raida in range(25):
    break(raida, source, [(dn, sn, pan)])  # SN may not exist!
```

**DO** find consensus SNs first:
```python
# CORRECT
all_sns = [get_available_sns(r, dn) for r in range(25)]
common = set.intersection(*[set(s) for s in all_sns])
```
