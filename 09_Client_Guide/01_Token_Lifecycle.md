# Token Lifecycle

## Overview

This document covers the complete lifecycle of a token from creation to transfer.

## Token States

```
                    FRACTURED
                        ^
                        |
                    (< 13 RAIDA)
                        |
UNKNOWN --> DETECT --> AUTHENTIC --> POWN --> TRANSFERRED
                        |                          |
                    (>= 13 RAIDA)                  |
                        |                          v
                        +<------------ (new owner detects)
```

## State Definitions

| State | Description | Action |
|-------|-------------|--------|
| UNKNOWN | Token not yet verified | Run Detect |
| AUTHENTIC | 13+ RAIDA confirm ownership | Can spend |
| FRACTURED | < 13 RAIDA confirm | Needs healing |
| TRANSFERRED | POWN succeeded, new owner | Update local AN |

## Operation: Detect (Verify Ownership)

```python
def detect_token(token) -> str:
    """
    Returns: 'authentic', 'fractured', 'counterfeit', or 'error'
    """
    results = []

    # Send to all 25 RAIDA in parallel
    for raida_id in range(25):
        try:
            success = send_detect(raida_id, token)
            results.append(success)
        except Exception:
            results.append(None)  # Network error

    # Count results
    passes = sum(1 for r in results if r is True)
    fails = sum(1 for r in results if r is False)
    errors = sum(1 for r in results if r is None)

    if passes >= 13:
        return 'authentic'
    elif fails >= 13:
        return 'counterfeit'
    elif passes > 0:
        return 'fractured'
    else:
        return 'error'
```

## Operation: POWN (Transfer Ownership)

```python
def transfer_token(token, recipient_pans: list) -> bool:
    """
    Transfer token to new owner.

    Args:
        token: Token with current ANs
        recipient_pans: 25 new ANs (one per RAIDA)

    Returns:
        True if consensus achieved (>= 13 success)
    """
    results = []

    # Send POWN to all 25 RAIDA
    for raida_id in range(25):
        try:
            success = send_pown(
                raida_id,
                token,
                recipient_pans[raida_id]
            )
            results.append((raida_id, success))
        except Exception:
            results.append((raida_id, None))

    # Count successes
    successes = sum(1 for _, r in results if r is True)

    if successes >= 13:
        # Update local token with new ANs where successful
        for raida_id, success in results:
            if success:
                token.an[raida_id] = recipient_pans[raida_id]
        return True
    else:
        # Transfer failed - token may be fractured
        return False
```

## Operation: Healing (Fix Fractured Token)

```python
def heal_token(token) -> bool:
    """
    Fix a fractured token using ticket-based healing.

    Requires: At least 13 RAIDA know the correct AN
    """
    # Step 1: Get tickets from working RAIDA
    tickets = []
    for raida_id in range(25):
        ticket = get_ticket(raida_id, token)
        if ticket:
            tickets.append((raida_id, ticket))

    if len(tickets) < 13:
        return False  # Not enough working RAIDA

    # Step 2: Find failed RAIDA
    detect_results = detect_all(token)
    failed_raida = [i for i, r in enumerate(detect_results) if not r]

    # Step 3: Fix each failed RAIDA
    for failed_id in failed_raida:
        # Collect tickets from 13 working RAIDA
        fix_tickets = tickets[:13]
        success = send_fix(failed_id, token, fix_tickets)

    # Step 4: Verify healing worked
    return detect_token(token) == 'authentic'
```

## Generating New ANs (PANs)

```python
import os

def generate_pans() -> list:
    """Generate 25 unique PANs for a transfer."""
    return [os.urandom(16) for _ in range(25)]
```

## Best Practices

### Always Detect Before Transfer
```python
# CORRECT
status = detect_token(token)
if status == 'authentic':
    transfer_token(token, new_pans)
elif status == 'fractured':
    heal_token(token)
    transfer_token(token, new_pans)
```

### Store Both Old and New ANs During Transfer
```python
# During transfer, keep backup
old_ans = token.an.copy()
new_ans = generate_pans()

success = transfer_token(token, new_ans)
if not success:
    # Restore old ANs for retry or healing
    token.an = old_ans
```

### Handle Partial Failures
```python
def safe_transfer(token, new_pans):
    results = pown_all_raida(token, new_pans)

    successes = sum(1 for r in results if r)
    if successes >= 13:
        # Success - update ANs
        for i, success in enumerate(results):
            if success:
                token.an[i] = new_pans[i]
            # Failed RAIDA still have old AN - will need healing
        return True
    elif successes > 0:
        # Partial - token is fractured
        # Some RAIDA have new AN, some have old
        token.status = 'fractured'
        return False
    else:
        # Complete failure - token unchanged
        return False
```
