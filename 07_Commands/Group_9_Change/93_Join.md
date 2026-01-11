# JOIN (Group 9, Code 93)

## Quick Reference

| Property | Value |
|----------|-------|
| Command Group | 9 |
| Command Code | 93 |
| Header Size | 32 bytes |
| Request Body | Variable |
| Response Body | New token + terminator |
| Encryption | Required |
| Transport | UDP preferred |

## Purpose

Joins smaller denomination tokens into one larger token.
Example: Join five 5-units into one 25-unit.

## Valid Join Combinations

| Sources | Target |
|---------|--------|
| 5×5 | 1×25 |
| 4×25 | 1×100 |
| 5×1 | 1×5 |
| 4×0.25 | 1×1 |

## Request Body

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0-15 | 16 | Challenge | 12 random + 4 CRC32 |
| 16 | 1 | Source Count | Number of source tokens |
| 17+ | 21*N | Sources | Source token records |
| +1 | 1 | Target DN | Target denomination |
| +4 | 4 | Target SN | Target serial number |
| +16 | 16 | Target PAN | New AN for target |
| Last 2 | 2 | Terminator | 0x3E 0x3E |

### Source Token Record (21 bytes each)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | Denomination |
| 1-4 | 4 | Serial Number (BE) |
| 5-20 | 16 | Current AN |

## Response

| Status | Body |
|--------|------|
| 250 (SUCCESS) | Confirmation |
| 160 (CHANGE_LIMIT) | Rate limit exceeded |
| 242 (FAIL) | Invalid sources or target |

## Algorithm

### Build Request
```pseudocode
function build_join_request(
    raida_id: int,
    source_tokens: list,
    target_dn: int,
    target_sn: int,
    target_pan: list  // 25 PANs
) -> bytes:
    source_count = len(source_tokens)
    body_len = 16 + 1 + (21 * source_count) + 1 + 4 + 16 + 2

    header = build_header_32(
        raida_id = raida_id,
        command_group = 9,
        command_code = 93,
        encryption_type = 1,
        key_dn = source_tokens[0].denomination,
        key_sn = source_tokens[0].serial_number,
        body_length = body_len
    )

    body = generate_challenge()
    body += bytes([source_count])

    for token in source_tokens:
        body += bytes([token.denomination])
        body += token.serial_number.to_bytes(4, 'big')
        body += token.an[raida_id]

    body += bytes([target_dn])
    body += target_sn.to_bytes(4, 'big')
    body += target_pan[raida_id]

    padded = pad_body(body, encryption_type=1)
    key = source_tokens[0].an[raida_id]
    encrypted = aes_ctr_encrypt(padded, key, nonce)

    return header + encrypted + b'\x3E\x3E'
```

### Complete Join Workflow
```pseudocode
function join_tokens(sources: list, target_dn: int) -> Token:
    // Step 1: Validate source values
    source_value = sum(DENOMINATION_VALUES[t.denomination] for t in sources)
    target_value = DENOMINATION_VALUES[target_dn]

    if source_value != target_value:
        raise Error(f"Value mismatch: {source_value} != {target_value}")

    // Step 2: Verify all source tokens are authentic
    for token in sources:
        if detect(token) != 'authentic':
            raise Error(f"Token {token.serial_number} not authentic")

    // Step 3: Get available SN for target denomination
    available_sns = {}
    for raida_id in range(25):
        sns = get_available_sns(raida_id, target_dn, sources[0])
        available_sns[raida_id] = sns

    common_sns = find_common_sns(available_sns, 1)
    if len(common_sns) < 1:
        raise Error("No available serial number for target")

    target_sn = common_sns[0]

    // Step 4: Generate PAN for new token
    target_pan = [os.urandom(16) for _ in range(25)]

    // Step 5: Execute join on all RAIDA
    results = []
    for raida_id in range(25):
        result = join_request(raida_id, sources, target_dn, target_sn, target_pan)
        results.append(result)

    // Step 6: Check consensus
    if sum(results) >= 13:
        return Token(target_dn, target_sn, target_pan)
    else:
        raise Error("Join failed - no consensus")
```

## Value Conservation

Join MUST preserve total value:

```python
# Valid: 5×5 = 25
join([token_5, token_5, token_5, token_5, token_5], target_dn=3)  # OK

# Invalid: 4×5 = 20 != 25
join([token_5, token_5, token_5, token_5], target_dn=3)  # FAIL
```

## Atomicity

Join is atomic per-RAIDA:
- Either all source tokens are consumed AND target created
- Or nothing changes

If consensus fails:
- Some RAIDA may have joined, others not
- Source tokens may be fractured
- Need healing to recover

## Common Mistakes

**DON'T** join tokens of wrong total value:
```python
# WRONG - 3×5=15 but target 25 needs 5×5=25
join([token_5, token_5, token_5], target_dn=3)
```

**DO** verify value before joining:
```python
# CORRECT
source_value = sum(values)
target_value = DENOMINATION_VALUES[target_dn]
assert source_value == target_value
join(sources, target_dn)
```

**DON'T** join without verifying sources:
```python
# WRONG - Source may be counterfeit
join(unverified_tokens, target_dn)
```

**DO** detect sources first:
```python
# CORRECT
for token in sources:
    assert detect(token) == 'authentic'
join(sources, target_dn)
```

**DON'T** ignore partial failures:
```python
# WRONG - May leave tokens fractured
results = [join_on_raida(r) for r in range(25)]
if sum(results) < 13:
    pass  # Ignoring failure!
```

**DO** handle partial failure with healing:
```python
# CORRECT
if sum(results) < 13:
    # Source tokens may be fractured
    for token in sources:
        heal_if_needed(token)
```
