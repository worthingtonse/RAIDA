# Status Codes

## Success Codes

| Code | Name | Description |
|------|------|-------------|
| 241 | ALL_PASS | All tokens authenticated |
| 242 | ALL_FAIL | All tokens failed |
| 243 | MIXED | Some passed, some failed (body has bitfield) |
| 250 | SUCCESS | Generic success |

## Header Errors (0-50)

| Code | Name | Description |
|------|------|-------------|
| 1 | INVALID_CLOUD_ID | Wrong cloud identifier |
| 2 | INVALID_SPLIT_ID | Invalid split field |
| 3 | INVALID_RAIDA_ID | RAIDA ID out of range |
| 4 | INVALID_SHARD_ID | Invalid shard field |
| 5 | INVALID_COMMAND_GROUP | Unknown command group |
| 6 | INVALID_COMMAND | Unknown command code |
| 7 | INVALID_TOKEN_ID | Unsupported token type |
| 16 | INVALID_PACKET_LENGTH | Packet too short |
| 17 | UDP_FRAME_TIMEOUT | UDP frame not received |
| 18 | WRONG_RAIDA | Request sent to wrong RAIDA |
| 25 | ENCRYPTION_TOKEN_NOT_FOUND | Cannot find decryption key |
| 27 | INVALID_ENCRYPTION_CODE | Unknown encryption type |
| 33 | INVALID_EOF | Missing terminator bytes |
| 37 | INVALID_CRC | Challenge CRC mismatch |
| 40 | INVALID_SN_OR_DENOMINATION | Token not found |

## Service Errors (100+)

| Code | Name | Description |
|------|------|-------------|
| 160 | CHANGE_LIMIT | Too many change requests |
| 179 | LOCKER_EMPTY | Locker not found or empty |
| 252 | INTERNAL_ERROR | Server internal error |
| 253 | NETWORK_ERROR | Network failure |
| 254 | MEMORY_ERROR | Memory allocation failed |

## Healing Status Codes (200+)

| Code | Name | Description |
|------|------|-------------|
| 208 | FIND_NEITHER | Neither AN nor PAN found |
| 209 | FIND_ALL_AN | All ANs matched |
| 210 | FIND_ALL_PAN | All PANs matched |
| 211 | FIND_MIXED | Mixed AN/PAN results |

## Retry Policy

| Code Range | Retryable | Action |
|------------|-----------|--------|
| 241-250 | No | Success, proceed |
| 1-50 | No | Fix request |
| 252-254 | Yes | Retry with backoff |
| Others | Depends | See specific code |
