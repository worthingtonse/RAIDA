# Key Derivation Specification

## Overview

This document specifies how encryption keys are derived for each encryption type.

## Type 1: AES-128 with Token AN

```
Key Size: 128 bits (16 bytes)
Source: Token authenticity number
```

The key is the 16-byte authenticity number (AN) of the token specified in the header.

### Key Lookup

From 32-byte header:
- Denomination: Byte 17
- Serial Number: Bytes 18-21 (big-endian)

### Implementation

```python
def get_key_type1(token, raida_id: int) -> bytes:
    """
    Get AES-128 key for Type 1 encryption.

    Args:
        token: The token whose AN is used as the key
        raida_id: The RAIDA server ID (0-24)

    Returns:
        16-byte AES key
    """
    return token.authenticity_numbers[raida_id]  # 16 bytes
```

---

## Type 2: AES-128 with Locker AN

```
Key Size: 128 bits (16 bytes)
Source: Locker authenticity number
```

Identical to Type 1, but the key is looked up in the locker index instead of the regular token storage.

### Key Lookup

From 32-byte header:
- Denomination: Byte 17
- Serial Number: Bytes 18-21 (big-endian)

### Implementation

```python
def get_key_type2(locker, raida_id: int) -> bytes:
    """
    Get AES-128 key for Type 2 encryption.

    Args:
        locker: The locker whose AN is used as the key
        raida_id: The RAIDA server ID (0-24)

    Returns:
        16-byte AES key
    """
    return locker.authenticity_numbers[raida_id]  # 16 bytes
```

---

## Type 3: Reserved

Type 3 is reserved for future use and is not currently implemented.

Servers will reject requests with encryption type 3.

---

## Summary Table

| Type | Key Size | Source | Status |
|------|----------|--------|--------|
| 0 | N/A | None (unencrypted) | Supported |
| 1 | 128-bit | Token AN | Supported |
| 2 | 128-bit | Locker AN | Supported |
| 3 | N/A | Reserved | Unsupported |
