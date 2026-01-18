# Error Recovery Guide

## Overview

This guide covers recovering from common error scenarios in RAIDA operations.

## Error Categories

| Category | Recoverable | Action |
|----------|-------------|--------|
| Network timeout | Yes | Retry with backoff |
| Authentication fail | No | Check AN correctness |
| Partial consensus | Sometimes | Healing may help |
| Rate limit | Yes | Wait and retry |

## Scenario 1: Network Timeout

### Symptoms
- Socket timeout exception
- No response from RAIDA

### Recovery
```python
def send_with_recovery(raida_id: int, packet: bytes) -> bytes:
    max_retries = 3
    base_timeout = 3000

    for attempt in range(max_retries):
        timeout = base_timeout * (1.5 ** attempt)
        try:
            return send_request(raida_id, packet, timeout)
        except TimeoutError:
            if attempt == max_retries - 1:
                raise
            continue
```

### Prevention
- Use adaptive timeouts based on RAIDA response history
- Implement per-RAIDA health tracking
- Skip consistently slow RAIDA in time-sensitive operations

## Scenario 2: Partial Consensus (Fractured Token)

### Symptoms
- Detect returns < 13 passes but > 0
- POWN succeeds on some RAIDA but not others

### Diagnosis
```python
def diagnose_token(token: Token) -> dict:
    results = detect_all_raida(token)

    passes = sum(1 for r in results if r == True)
    fails = sum(1 for r in results if r == False)
    errors = sum(1 for r in results if r is None)

    if passes >= 13:
        return {'status': 'authentic', 'healthy': True}
    elif passes > 0:
        return {
            'status': 'fractured',
            'healthy': False,
            'working_raida': [i for i, r in enumerate(results) if r],
            'broken_raida': [i for i, r in enumerate(results) if not r]
        }
    else:
        return {'status': 'counterfeit_or_error', 'healthy': False}
```

### Recovery: Healing
```python
def heal_fractured_token(token: Token) -> bool:
    diagnosis = diagnose_token(token)

    if diagnosis['status'] != 'fractured':
        return False

    working = diagnosis['working_raida']
    broken = diagnosis['broken_raida']

    if len(working) < 13:
        return False  # Not enough to heal

    # Get tickets from working RAIDA
    tickets = []
    for raida_id in working[:13]:
        ticket = get_ticket(raida_id, token)
        if ticket:
            tickets.append((raida_id, ticket))

    # Fix broken RAIDA
    for raida_id in broken:
        fix(raida_id, token, tickets)

    # Verify
    new_diagnosis = diagnose_token(token)
    return new_diagnosis['status'] == 'authentic'
```

## Scenario 3: Failed Transfer (POWN)

### Symptoms
- POWN returned mixed results
- Token ownership uncertain

### Recovery
```python
def recover_failed_pown(
    token: Token,
    attempted_pans: list
) -> str:
    # Check current state with Find
    states = {}
    for raida_id in range(25):
        result = find(raida_id, token, attempted_pans[raida_id])
        states[raida_id] = result

    has_an = [r for r, s in states.items() if s == 'has_an']
    has_pan = [r for r, s in states.items() if s == 'has_pan']

    if len(has_pan) >= 13:
        # Transfer mostly succeeded - complete it
        for raida_id in has_an:
            pown(raida_id, token, attempted_pans)
        # Update local token
        for raida_id in has_pan:
            token.an[raida_id] = attempted_pans[raida_id]
        return 'completed_transfer'

    elif len(has_an) >= 13:
        # Transfer mostly failed - keep old ownership
        # Heal RAIDA that have PAN back to AN
        return 'kept_original'

    else:
        # Truly fractured - need manual intervention
        return 'fractured_needs_healing'
```

## Scenario 4: Rate Limit (Status 160)

### Symptoms
- Status code 160 on change operations
- GetAvailableSNs, Break, or Join fails

### Recovery
```python
def change_with_rate_limit_handling(operation, *args):
    max_wait = 300  # 5 minutes max
    wait_time = 10  # Start with 10 seconds

    while wait_time <= max_wait:
        try:
            return operation(*args)
        except RateLimitError:
            time.sleep(wait_time)
            wait_time *= 2  # Exponential backoff

    raise Error("Rate limit not cleared after maximum wait")
```

## Scenario 5: Encryption Key Not Found (Status 25)

### Symptoms
- Status code 25
- RAIDA cannot decrypt request

### Causes
1. Wrong token used for encryption key
2. Token AN out of sync with RAIDA
3. Token not registered on this RAIDA

### Recovery
```python
def handle_key_not_found(token: Token, raida_id: int):
    # First, verify token is authentic on this RAIDA
    result = detect(raida_id, token)

    if result == False:
        # Token not authentic - may need healing
        return 'need_healing'

    if result is None:
        # Network error
        return 'retry'

    # If detect passes but encrypt fails, check key selection
    return 'check_key_token_selection'
```

## Error Status Code Quick Reference

| Code | Name | Retry? | Recovery Action |
|------|------|--------|-----------------|
| 25 | KEY_NOT_FOUND | No | Check encryption key |
| 33 | INVALID_EOF | No | Check terminator |
| 37 | INVALID_CRC | No | Fix CRC calculation |
| 160 | CHANGE_LIMIT | Yes | Wait and retry |
| 179 | LOCKER_EMPTY | No | Check locker ID/code |
| 241 | ALL_PASS | - | Success |
| 242 | ALL_FAIL | No | Check token authenticity |
| 243 | MIXED | - | Parse bitfield |
| 252 | INTERNAL_ERROR | Yes | Retry with backoff |
| 253 | NETWORK_ERROR | Yes | Retry |

## Best Practices

1. **Always track operation state**: Before retrying, know what succeeded
2. **Use idempotent operations**: Detect is safe to retry; POWN is not
3. **Implement circuit breakers**: Stop retrying after consistent failures
4. **Log everything**: Diagnosis requires complete operation history
5. **Handle partial success**: Most operations can partially complete
