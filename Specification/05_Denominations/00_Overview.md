# Denominations

## Core Formula

The denomination system uses a **logarithmic encoding** where the denomination code is a power-of-10 exponent:

```
Token Value = 10 ^ denomination_code
```

The denomination code is a **signed 8-bit integer (int8_t)**. When transmitted over the protocol, negative values use two's complement encoding.

### Quick Examples

| Code | Calculation | Value |
|------|-------------|-------|
| -8   | 10^-8       | 0.00000001 |
| -2   | 10^-2       | 0.01 |
| 0    | 10^0        | 1 |
| 3    | 10^3        | 1,000 |
| 6    | 10^6        | 1,000,000 |

## Byte Encoding

| Signed Code | Hex Byte | Unsigned Value |
|-------------|----------|----------------|
| -8          | 0xF8     | 248            |
| -1          | 0xFF     | 255            |
| 0           | 0x00     | 0              |
| 6           | 0x06     | 6              |

**AI Note:** Parse the byte as uint8_t, then cast to int8_t to get the signed exponent. Do NOT interpret 0xF8 as 248; it represents -8.

## Master Denomination Table

The smallest unit is the **Satoshi** (also called **Nano**). All values can be expressed as integer multiples of Satoshis, eliminating floating-point ambiguity.

### Fractional Denominations (Codes -8 to -1)

| Name | Alias | Value (CC) | Value (Satoshis) | Code | Hex |
|------|-------|------------|------------------|------|-----|
| 1-Satoshi | Nano | 0.00000001 | 1 | -8 | 0xF8 |
| 10-Satoshi | | 0.0000001 | 10 | -7 | 0xF9 |
| 100-Satoshi | Micro | 0.000001 | 100 | -6 | 0xFA |
| 1k-Satoshi | | 0.00001 | 1,000 | -5 | 0xFB |
| 10k-Satoshi | Tick | 0.0001 | 10,000 | -4 | 0xFC |
| 100k-Satoshi | Mil | 0.001 | 100,000 | -3 | 0xFD |
| 1M-Satoshi | Cent | 0.01 | 1,000,000 | -2 | 0xFE |
| 10M-Satoshi | Dime | 0.1 | 10,000,000 | -1 | 0xFF |

### Whole Denominations (Codes 0 to 6)

| Name | Alias | Value (CC) | Value (Satoshis) | Code | Hex |
|------|-------|------------|------------------|------|-----|
| One | 1-CC | 1 | 100,000,000 | 0 | 0x00 |
| Ten | 10-CC | 10 | 1,000,000,000 | 1 | 0x01 |
| Hundred | 100-CC | 100 | 10,000,000,000 | 2 | 0x02 |
| Thousand | 1k-CC | 1,000 | 100,000,000,000 | 3 | 0x03 |
| Ten-K | 10k-CC | 10,000 | 1,000,000,000,000 | 4 | 0x04 |
| Hundred-K | 100k-CC | 100,000 | 10,000,000,000,000 | 5 | 0x05 |
| Million | 1M-CC | 1,000,000 | 100,000,000,000,000 | 6 | 0x06 |

**Limits:**
- **1-Satoshi (code -8)** is the smallest unit - cannot be broken further
- **Million (code 6)** is the largest unit - cannot be joined further

## Break and Join Rules

Denominations can be exchanged at a 10:1 ratio:

### BREAK (1 large → 10 smaller)
```
1 token of code N  →  10 tokens of code (N-1)
```

Examples:
- 1 One (code 0) → 10 Dimes (code -1)
- 1 Thousand (code 3) → 10 Hundreds (code 2)

### JOIN (10 smaller → 1 larger)
```
10 tokens of code N  →  1 token of code (N+1)
```

Examples:
- 10 Cents (code -2) → 1 Dime (code -1)
- 10 Ones (code 0) → 1 Ten (code 1)

## Conversion Formula

To convert quantity Q from denomination A to denomination B:

```
new_quantity = Q × 10^(A - B)
```

**Example:** Convert 500 Cents to Ones
- Cents = code -2, Ones = code 0
- new_quantity = 500 × 10^(-2 - 0) = 500 × 0.01 = 5 Ones

## Protocol Encoding

In the RAIDA protocol, a coin is identified by 5 bytes:

```
[Denomination 1B][Serial Number 4B]
     int8_t          uint32_t BE
```

| Coin | Denomination | Serial | Bytes (Hex) |
|------|--------------|--------|-------------|
| 1-Satoshi #1234 | -8 (0xF8) | 1234 | F8 00 00 04 D2 |
| Cent #5000 | -2 (0xFE) | 5000 | FE 00 00 13 88 |
| One #999 | 0 (0x00) | 999 | 00 00 00 03 E7 |
| Million #1 | 6 (0x06) | 1 | 06 00 00 00 01 |

## Design Rationale

**Why logarithmic encoding?**
A single signed byte can represent values spanning 14 orders of magnitude (10^-8 to 10^6). Linear encoding would require many more bytes.

**Why base 10?**
Metric system compatibility. Conversions are trivial: just add/subtract denomination codes. 10 tokens of code N always equals 1 token of code N+1.

**Why signed integer?**
Unifies fractional (negative exponents) and whole (positive exponents) values in one coherent system. Zero represents the base unit (1 CC).

**Why Satoshi as base name?**
Named after the concept of an indivisible smallest unit. This is NOT related to Bitcoin - it simply denotes the smallest possible token value in this system.

## AI Implementation Guide

### Parse Denomination Byte

```python
def parse_denomination(byte_value: int) -> int:
    """Convert unsigned byte to signed denomination code."""
    if byte_value > 127:
        return byte_value - 256  # Two's complement
    return byte_value

# parse_denomination(0xF8) returns -8
# parse_denomination(0x00) returns 0
```

### Compute Token Value

```python
def get_token_value(denomination_code: int) -> float:
    """Get token value from denomination code."""
    return 10 ** denomination_code

# get_token_value(-8) returns 0.00000001
# get_token_value(0)  returns 1.0
# get_token_value(6)  returns 1000000.0
```

### Compute Value in Satoshis (Integer)

```python
def get_satoshi_value(denomination_code: int) -> int:
    """Get token value in Satoshis (avoids floating-point)."""
    return 10 ** (denomination_code + 8)

# get_satoshi_value(-8) returns 1
# get_satoshi_value(-2) returns 1000000
# get_satoshi_value(0)  returns 100000000
```

### Compute Wallet Total

```python
def compute_wallet_value(coins: list) -> int:
    """Compute total wallet value in Satoshis."""
    total = 0
    for denom_code, count in coins:
        total += count * (10 ** (denom_code + 8))
    return total

# Example: 5 Ones + 3 Dimes + 50 Cents
# compute_wallet_value([(0, 5), (-1, 3), (-2, 50)])
# = 5×100000000 + 3×10000000 + 50×1000000 = 580000000 Satoshis
```

## Common AI Mistakes to Avoid

1. **DO NOT** treat 0xF8 as 248. It is -8 (signed two's complement).

2. **DO NOT** confuse code with value:
   - Code -2 does NOT mean "negative 2 coins"
   - It means value = 10^-2 = 0.01

3. **DO NOT** assume linear relationship:
   - Code 2 is NOT 2× the value of Code 1
   - Code 2 = 100, Code 1 = 10 (10× relationship)

4. **DO NOT** invent denomination names. Use the table above as the single source of truth.

5. **DO** use integer Satoshi values for calculations to avoid floating-point errors.
