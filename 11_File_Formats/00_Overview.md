# File Formats Overview

## Token Storage Formats

RAIDA clients need to store tokens locally. Several formats are supported.

## Format Comparison

| Format | Extension | Use Case | Human Readable |
|--------|-----------|----------|----------------|
| Binary | `.bin` | Compact storage | No |
| JSON | `.json` | Development, debugging | Yes |
| CSV | `.csv` | Spreadsheet export | Yes |
| Stack | `.stack` | Batch of same denomination | No |

## Binary Format (.bin)

Most compact format for single token.

### Structure

```
Offset  Size  Field
0       1     Denomination
1       4     Serial Number (big-endian)
5       400   ANs (25 × 16 bytes)
---
Total: 405 bytes
```

### Reading Binary Token

```python
def read_binary_token(filepath: str) -> dict:
    with open(filepath, 'rb') as f:
        data = f.read()

    if len(data) != 405:
        raise ValueError(f"Invalid size: {len(data)}")

    return {
        'denomination': data[0],
        'serial_number': int.from_bytes(data[1:5], 'big'),
        'ans': [data[5 + i*16 : 5 + (i+1)*16] for i in range(25)]
    }
```

### Writing Binary Token

```python
def write_binary_token(filepath: str, token: dict):
    data = bytearray()
    data.append(token['denomination'])
    data.extend(token['serial_number'].to_bytes(4, 'big'))
    for an in token['ans']:
        data.extend(an)

    with open(filepath, 'wb') as f:
        f.write(data)
```

## JSON Format (.json)

Human-readable format for development.

### Structure

```json
{
  "denomination": 1,
  "serial_number": 102205,
  "ans": [
    "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
    "b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5",
    ...
  ]
}
```

### Reading JSON Token

```python
import json

def read_json_token(filepath: str) -> dict:
    with open(filepath, 'r') as f:
        data = json.load(f)

    return {
        'denomination': data['denomination'],
        'serial_number': data['serial_number'],
        'ans': [bytes.fromhex(an) for an in data['ans']]
    }
```

## Stack Format (.stack)

Multiple tokens of same denomination.

### Structure

```
Offset  Size       Field
0       1          Denomination
1       4          Token count (N)
5       N × 404    Token records
```

### Token Record (404 bytes)

```
Offset  Size  Field
0       4     Serial Number
4       400   ANs (25 × 16)
```

## Wallet Directory Structure

```
wallet/
├── tokens/
│   ├── 1/           # Denomination 1
│   │   ├── 102205.bin
│   │   └── 102206.bin
│   ├── 5/           # Denomination 5
│   │   └── 50001.bin
│   └── 25/          # Denomination 25
│       └── 25001.bin
├── exported/        # Tokens being sent
├── imported/        # Tokens being received
└── config.json      # Wallet configuration
```

## Security Considerations

1. **File Permissions**: Token files contain secrets - restrict access
2. **Encryption at Rest**: Consider encrypting wallet directory
3. **Backup**: ANs are the only proof of ownership - backup regularly
4. **Never Share**: ANs should never be transmitted except during POWN
