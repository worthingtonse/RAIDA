# Healing Walkthrough

## Overview

This guide provides a complete walkthrough of healing a fractured token.

## What is a Fractured Token?

A token is fractured when:
- Fewer than 13 RAIDA recognize its AN
- But more than 0 RAIDA recognize it

This typically happens when:
- POWN partially completed
- Network failure during transfer
- RAIDA temporarily unavailable

## Prerequisites

To heal a token, you need:
1. The token must be authentic on at least 13 RAIDA
2. Access to those 13+ RAIDA
3. The correct AN (which working RAIDA have)

## Step-by-Step Healing Process

### Step 1: Diagnose the Token

```python
def diagnose(token: Token) -> dict:
    """Determine token health status."""
    results = []

    for raida_id in range(25):
        try:
            success = detect(raida_id, token)
            results.append((raida_id, success))
        except Exception as e:
            results.append((raida_id, None))

    working = [r for r, s in results if s is True]
    broken = [r for r, s in results if s is False]
    errors = [r for r, s in results if s is None]

    return {
        'working_raida': working,
        'broken_raida': broken,
        'error_raida': errors,
        'can_heal': len(working) >= 13,
        'is_healthy': len(working) >= 13 and len(broken) == 0
    }
```

**Example output:**
```python
{
    'working_raida': [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12],
    'broken_raida': [13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24],
    'error_raida': [],
    'can_heal': True,
    'is_healthy': False
}
```

### Step 2: Get Tickets from Working RAIDA

```python
def get_healing_tickets(token: Token, working_raida: list) -> list:
    """Get tickets from RAIDA that know the token."""
    tickets = []

    # Need at least 13 tickets
    for raida_id in working_raida[:13]:
        try:
            ticket = get_ticket(raida_id, token)
            if ticket:
                tickets.append((raida_id, ticket))
                print(f"Got ticket from RAIDA {raida_id}")
        except Exception as e:
            print(f"Failed to get ticket from RAIDA {raida_id}: {e}")

    return tickets
```

**What GetTicket does:**
1. Sends token DN, SN, and AN to RAIDA
2. RAIDA verifies the AN matches its records
3. RAIDA returns a time-limited ticket (opaque 16 bytes)
4. Ticket proves you owned the token at this RAIDA

### Step 3: Fix Each Broken RAIDA

```python
def fix_broken_raida(
    token: Token,
    broken_raida: list,
    tickets: list
) -> dict:
    """Send Fix command to each broken RAIDA."""
    results = {}

    for raida_id in broken_raida:
        try:
            success = fix(raida_id, token, tickets)
            results[raida_id] = success
            print(f"RAIDA {raida_id}: {'Fixed' if success else 'Failed'}")
        except Exception as e:
            results[raida_id] = False
            print(f"RAIDA {raida_id}: Error - {e}")

    return results
```

**What Fix does:**
1. Sends token DN, SN, AN, and 13+ tickets
2. Broken RAIDA validates each ticket with source RAIDA
3. If 13+ tickets validate, RAIDA stores the AN
4. Token now authenticates on this RAIDA

### Step 4: Verify Healing Success

```python
def verify_healing(token: Token) -> bool:
    """Check if token is now healthy."""
    diagnosis = diagnose(token)

    if diagnosis['is_healthy']:
        print("Token fully healed!")
        return True
    elif len(diagnosis['working_raida']) >= 13:
        print(f"Token usable but {len(diagnosis['broken_raida'])} RAIDA still broken")
        return True
    else:
        print("Healing failed - token still fractured")
        return False
```

## Complete Healing Function

```python
def heal_token(token: Token) -> bool:
    """
    Complete healing workflow.

    Returns True if token is now usable (13+ RAIDA working).
    """
    print(f"Starting healing for token SN={token.serial_number}")

    # Step 1: Diagnose
    print("\n--- Step 1: Diagnosing token ---")
    diagnosis = diagnose(token)

    if diagnosis['is_healthy']:
        print("Token is already healthy!")
        return True

    if not diagnosis['can_heal']:
        print(f"Cannot heal: only {len(diagnosis['working_raida'])} RAIDA working")
        print("Need at least 13 working RAIDA to heal")
        return False

    print(f"Working RAIDA: {diagnosis['working_raida']}")
    print(f"Broken RAIDA: {diagnosis['broken_raida']}")

    # Step 2: Get tickets
    print("\n--- Step 2: Getting tickets ---")
    tickets = get_healing_tickets(token, diagnosis['working_raida'])

    if len(tickets) < 13:
        print(f"Only got {len(tickets)} tickets, need 13")
        return False

    print(f"Got {len(tickets)} tickets")

    # Step 3: Fix broken RAIDA
    print("\n--- Step 3: Fixing broken RAIDA ---")
    fix_results = fix_broken_raida(token, diagnosis['broken_raida'], tickets)

    fixed = sum(1 for r in fix_results.values() if r)
    print(f"Fixed {fixed}/{len(diagnosis['broken_raida'])} RAIDA")

    # Step 4: Verify
    print("\n--- Step 4: Verifying ---")
    return verify_healing(token)
```

## Example Session

```
Starting healing for token SN=102205

--- Step 1: Diagnosing token ---
Working RAIDA: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
Broken RAIDA: [13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24]

--- Step 2: Getting tickets ---
Got ticket from RAIDA 0
Got ticket from RAIDA 1
...
Got ticket from RAIDA 12
Got 13 tickets

--- Step 3: Fixing broken RAIDA ---
RAIDA 13: Fixed
RAIDA 14: Fixed
...
RAIDA 24: Fixed
Fixed 12/12 RAIDA

--- Step 4: Verifying ---
Token fully healed!
```

## Common Issues

### Not Enough Working RAIDA
```
Cannot heal: only 10 RAIDA working
```
**Solution**: Wait for network issues to resolve, or token may be unrecoverable.

### Tickets Expired
```
RAIDA 15: Failed (tickets expired)
```
**Solution**: Get fresh tickets immediately before fixing.

### Inconsistent State
```
Some RAIDA have different AN
```
**Solution**: Use Find command to diagnose, may need manual intervention.

## Prevention

Avoid fractured tokens by:
1. Always verify POWN consensus before updating local AN
2. Keep old AN until new AN has 13+ consensus
3. Detect tokens periodically to catch issues early
