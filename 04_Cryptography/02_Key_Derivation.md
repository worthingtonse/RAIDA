# Key Derivation Specification

## CRITICAL SECURITY NOTICE

AES-256 encryption types (4, 5, 7) MUST use true 256-bit keys.
Duplicating a 128-bit value to create a 256-bit key is NOT acceptable
as it reduces entropy to 128 bits.

## Type 1: AES-128 with Token AN

```
Key Size: 128 bits (16 bytes)
Source: Token authenticity number

Key = AN[raida_id]
```

Where `AN[raida_id]` is the 16-byte authenticity number for the
specific RAIDA being contacted.

### Implementation
```python
def get_key_type1(token, raida_id: int) -> bytes:
    return token.authenticity_numbers[raida_id]  # 16 bytes
```

---

## Type 4: AES-256 with 256-bit Shared Secret

```
Key Size: 256 bits (32 bytes)
Source: True 256-bit shared secret
```

### Option A: Key-Type Tokens (Recommended)

Tokens with special denominations (0x07-0x0B) store 32-byte secrets:

```python
def get_key_type4_keytokens(key_token, raida_id: int) -> bytes:
    # Key tokens have 32-byte secrets instead of 16-byte ANs
    return key_token.secret[raida_id]  # 32 bytes
```

### Option B: Derived from Token AN (If using currency tokens)

If a 256-bit key is needed but only a 16-byte AN is available,
derive using SHA-256:

```python
import hashlib

def get_key_type4_derived(token, raida_id: int) -> bytes:
    an = token.authenticity_numbers[raida_id]  # 16 bytes
    # Derive 256-bit key using SHA-256
    return hashlib.sha256(an).digest()  # 32 bytes
```

### INCORRECT Implementation (DO NOT USE)

```python
# WRONG - Only 128 bits of entropy!
def get_key_type4_WRONG(token, raida_id: int) -> bytes:
    an = token.authenticity_numbers[raida_id]
    return an + an  # Duplicating reduces security!
```

---

## Type 5: AES-256 with Two Token ANs

```
Key Size: 256 bits (32 bytes)
Source: Concatenation of two 16-byte ANs
```

### Key Construction

```
Key = AN1 || AN2

Where:
  AN1 = First token's AN (bytes 0-15 of key)
  AN2 = Second token's AN (bytes 16-31 of key)
```

### Token Identification

From 48-byte header:
- Token 1: Denomination at byte 17, SN at bytes 18-21
- Token 2: Denomination at byte 24, SN at bytes 25-28

### Implementation

```python
def get_key_type5(token1, token2, raida_id: int) -> bytes:
    an1 = token1.authenticity_numbers[raida_id]  # 16 bytes
    an2 = token2.authenticity_numbers[raida_id]  # 16 bytes
    return an1 + an2  # 32 bytes total, 256 bits entropy
```

---

## Type 7: RAIDA-to-RAIDA Secret

```
Key Size: 256 bits (32 bytes)
Source: Pre-shared secret between RAIDA servers
```

Used for inter-RAIDA communication (healing, synchronization).

### Implementation

```python
def get_key_type7(source_raida: int, dest_raida: int, key_id: int) -> bytes:
    # Look up pre-shared secret from secure storage
    return raida_secrets[source_raida][dest_raida][key_id]  # 32 bytes
```

---

## Summary Table

| Type | Key Size | Entropy | Source |
|------|----------|---------|--------|
| 1 | 128-bit | 128-bit | Token AN |
| 4 | 256-bit | 256-bit | 32-byte secret OR SHA-256(AN) |
| 5 | 256-bit | 256-bit | AN1 \|\| AN2 |
| 7 | 256-bit | 256-bit | RAIDA pre-shared secret |
