# Status Codes

## Overview

Status codes are returned in byte 2 of the response header. They indicate the result of the command execution.

## Success Codes (241-250)

| Code | Name | Description |
|------|------|-------------|
| 241 | ALL_PASS | All tokens in request passed authentication |
| 242 | ALL_FAIL | All tokens in request failed authentication |
| 243 | MIXED | Some passed, some failed (body contains bitfield) |
| 250 | SUCCESS | Generic success |

## No Error

| Code | Name | Description |
|------|------|-------------|
| 0 | NO_ERROR | Request processed successfully |

## Header/Routing Errors (1-20)

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 1 | INVALID_CLOUD_ID | Wrong cloud identifier | No |
| 2 | INVALID_SPLIT_ID | Invalid split field | No |
| 3 | INVALID_RAIDA_ID | RAIDA ID out of range (0-24) | No |
| 4 | INVALID_SHARD_ID | Invalid shard field | No |
| 5 | INVALID_COMMAND_GROUP | Unknown command group | No |
| 6 | INVALID_COMMAND | Unknown command code | No |
| 7 | INVALID_COIN_ID | Unsupported coin/token type | No |
| 8 | COIN_NOT_FOUND | Coin not found in database | No |
| 15 | INVALID_UDP_FRAME_COUNT | Invalid UDP frame count | No |
| 16 | INVALID_PACKET_LENGTH | Packet too short | No |
| 17 | UDP_FRAME_TIMEOUT | UDP frame not received in time | Yes |
| 18 | WRONG_RAIDA | Request sent to wrong RAIDA | No |
| 20 | SHARD_NOT_AVAILABLE | Requested shard not available | Yes |

## Encryption Errors (25-40)

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 25 | ENCRYPTION_COIN_NOT_FOUND | Encryption key coin not found | No |
| 27 | INVALID_ENCRYPTION_CODE | Unknown encryption type | No |
| 33 | INVALID_EOF | Missing terminator bytes (0x3E3E) | No |
| 34 | INVALID_ENCRYPTION | Encryption validation failed | No |
| 36 | EMPTY_REQUEST | Empty request body | No |
| 37 | INVALID_CRC | Challenge CRC mismatch | No |
| 38 | ADMIN_AUTH | Admin authentication failed | No |
| 39 | COINS_NOT_DIV | Coins not divisible for change | No |
| 40 | INVALID_SN_OR_DENOMINATION | Invalid serial number or denomination | No |

## Page/Storage Errors (41-48)

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 41 | PAGE_IS_NOT_RESERVED | Page not reserved for operation | No |
| 42 | NO_TICKET_SLOT | No ticket slot available | Yes |
| 43 | NO_TICKET_FOUND | Ticket not found | No |
| 44 | TICKET_CLAIMED_ALREADY | Ticket already claimed by another RAIDA | No |
| 45 | TOO_MANY_COINS | Too many coins in single request | No |
| 46 | INVALID_SHARD | Invalid shard specified | No |
| 47 | DELETE_COINS | Error deleting coins | Yes |
| 48 | LEGACY_DB | Legacy database error | Yes |

## Crossover/Trade Errors (49-52)

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 49 | CROSSOVER_FULL | Crossover queue is full | Yes |
| 50 | INVALID_TRADE_COIN | Invalid trade coin | No |
| 51 | TRADE_LOCKER_NOT_FOUND | Trade locker not found | No |
| 52 | NO_PRIVATE_KEY | No private key available | No |

## Command Errors (89-90)

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 89 | NOT_IMPLEMENTED | Command not implemented | No |
| 90 | BAD_COINS | Bad coins in request | No |

## Trade/Locker Errors (148-160)

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 148 | TRADE_LOCKER_EXISTS | Trade locker already exists | No |
| 149 | NO_TRADE_LOCKER | No trade locker found | No |
| 150 | WAITING | Operation waiting/pending | Yes |
| 152 | NO_BTC_IN_WALLET | No BTC in wallet | No |
| 153 | FEW_COINS_IN_LOCKER | Too few coins in locker | No |
| 154 | LOCKER_USED | Locker already in use | No |
| 160 | REQUEST_RATE | Request rate limit exceeded | Yes |

## Payment Errors (167-169)

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 167 | PAYMENT_PROCESSING | Payment still processing | Yes |
| 168 | PAYMENT_INSUFFICIENT | Insufficient payment amount | No |
| 169 | PAYMENT_REQUIRED | Payment required for operation | No |

## Transaction/External Errors (177-193)

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 177 | TXN_PROCESSED | Transaction already processed | No |
| 178 | CRYPTO_CONNECT | Crypto service connection error | Yes |
| 179 | LOCKER_EMPTY_OR_NOT_EXISTS | Locker empty or doesn't exist | No |
| 180 | PROXY_CONNECT | Proxy connection error | Yes |
| 181 | PRICE | Price calculation error | Yes |
| 182 | NO_COINS | No coins available | No |
| 183 | TX_SEEN | Transaction seen (pending) | Yes |
| 184 | NXRECORD | Record not found | No |
| 185 | NXDOMAIN | Domain not found | No |
| 186 | UNKNOWN | Unknown error | Yes |
| 187 | PROXY | Proxy error | Yes |
| 188 | KEY_BUILD | Key build error | Yes |
| 189 | EXTERNAL_BACKEND | External backend error | Yes |
| 190 | TX_EMPTY | Transaction empty | No |
| 191 | TX_NOT_EXIST | Transaction doesn't exist | No |
| 192 | AMOUNT_MISMATCH | Amount mismatch | No |
| 193 | NO_ENTRY | No entry found | No |

## Filesystem/Key Errors (194-205)

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 194 | FILESYSTEM | Filesystem error | Yes |
| 195 | INVALID_KEY_START | Invalid key start position | No |
| 196 | INVALID_KEY_LENGTH | Invalid key length | No |
| 197 | COIN_LOAD | Coin load error | Yes |
| 198 | INVALID_PARAMETER | Invalid parameter | No |
| 199 | INVALID_PAN | Invalid PAN (proposed AN) | No |
| 200 | INVALID_AN | Invalid AN (authenticity number) | No |
| 201 | FILE_EXISTS | File already exists | No |
| 202 | FILE_NOT_EXIST | File doesn't exist | No |
| 203 | INVALID_TRANSACTION | Invalid transaction format | No |
| 204 | BLOCKCHAIN | Blockchain error | Yes |
| 205 | ASSEMBLE | Assembly error | Yes |
| 206 | RAIDA_TIMEOUT | RAIDA timeout during Fix | Yes |
| 207 | RAIDA_CONNECTION | RAIDA connection error / Invalid HMAC | Yes |

## Find Service Codes (208-211)

| Code | Name | Description |
|------|------|-------------|
| 208 | FIND_NEITHER | Neither AN nor PAN matched |
| 209 | FIND_ALL_AN | All coins matched current AN |
| 210 | FIND_ALL_PAN | All coins matched PAN |
| 211 | FIND_MIXED | Mixed AN/PAN results (see body) |

## Key/Secret Errors (212-216)

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 212 | SECRET_NOT_FOUND | Secret not found | No |
| 213 | INVALID_CS_ID | Invalid CS ID | No |
| 214 | KEY_DERIVATION_FAILED | Key derivation failed | No |
| 215 | INVALID_KEY_ID | Invalid key ID | No |
| 216 | KEY_NOT_FOUND | Key not found | No |

## System Errors (252-255)

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 252 | INTERNAL | Internal server error | Yes |
| 253 | NETWORK | Network error | Yes |
| 254 | MEMORY_ALLOC | Memory allocation failed | Yes |
| 255 | INVALID_ROUTING | Invalid routing | No |

## Retry Policy Summary

| Code Range | General Guidance |
|------------|-----------------|
| 0, 241-250 | Success - do not retry |
| 1-50 | Request error - fix request, do not retry |
| 150, 167, 183 | Pending - retry after delay |
| 160 | Rate limited - retry with exponential backoff |
| 178-189, 252-254 | Transient error - retry with backoff |
| Other | Check specific code |

## Handling MIXED Results (243)

When status code 243 (MIXED) is returned, the response body contains a bitfield indicating which tokens passed and which failed:

```python
def parse_mixed_results(body: bytes, token_count: int) -> list:
    """
    Parse bitfield from MIXED response.

    Returns list of booleans: True=pass, False=fail
    """
    results = []
    for i in range(token_count):
        byte_index = i // 8
        bit_index = 7 - (i % 8)  # MSB first

        if byte_index < len(body):
            bit = (body[byte_index] >> bit_index) & 1
            results.append(bit == 1)
        else:
            results.append(False)

    return results
```
