# Logging Guide

## Overview

Proper logging is essential for debugging RAIDA client issues.
This guide specifies what to log and how to protect sensitive data.

## Log Levels

| Level | Use Case |
|-------|----------|
| DEBUG | Packet hex dumps, timing details |
| INFO | Operations, consensus results |
| WARNING | Retries, partial failures |
| ERROR | Failures, exceptions |

## What to Log

### Always Log

| Event | Fields |
|-------|--------|
| Request sent | RAIDA ID, command, timestamp |
| Response received | RAIDA ID, status code, latency |
| Timeout | RAIDA ID, timeout value |
| Consensus result | Pass count, fail count, outcome |
| Operation complete | Operation type, success/failure |

### Debug Level Only

| Event | Fields |
|-------|--------|
| Packet headers | Hex dump (NOT body) |
| Encryption details | Type, key source (NOT key value) |
| Retry attempts | Attempt number, new timeout |

## What to NEVER Log

| Data | Reason |
|------|--------|
| Authenticity Numbers (AN) | Token ownership secret |
| Proposed ANs (PAN) | Future ownership secret |
| Encryption keys | Security compromise |
| Full packet bodies | May contain secrets |

## Structured Logging Format

```python
import logging
import json
from datetime import datetime

class RAIDALogger:
    def __init__(self, name: str):
        self.logger = logging.getLogger(name)

    def log_request(self, raida_id: int, command: str, token_count: int):
        self.logger.info(json.dumps({
            'event': 'request_sent',
            'timestamp': datetime.utcnow().isoformat(),
            'raida_id': raida_id,
            'command': command,
            'token_count': token_count
        }))

    def log_response(self, raida_id: int, status: int, latency_ms: float):
        self.logger.info(json.dumps({
            'event': 'response_received',
            'timestamp': datetime.utcnow().isoformat(),
            'raida_id': raida_id,
            'status': status,
            'status_name': STATUS_NAMES.get(status, 'UNKNOWN'),
            'latency_ms': round(latency_ms, 2)
        }))

    def log_consensus(self, operation: str, results: list):
        passes = sum(1 for r in results if r)
        fails = len(results) - passes

        self.logger.info(json.dumps({
            'event': 'consensus_result',
            'timestamp': datetime.utcnow().isoformat(),
            'operation': operation,
            'passes': passes,
            'fails': fails,
            'consensus_achieved': passes >= 13
        }))
```

## Log Redaction

```python
def redact_sensitive(data: dict) -> dict:
    """Remove sensitive fields before logging."""
    sensitive_keys = {'an', 'pan', 'key', 'authenticity_number',
                      'proposed_an', 'encryption_key', 'secret'}

    result = {}
    for key, value in data.items():
        if key.lower() in sensitive_keys:
            result[key] = '[REDACTED]'
        elif isinstance(value, dict):
            result[key] = redact_sensitive(value)
        elif isinstance(value, bytes):
            # Only show length, not content
            result[key] = f'<{len(value)} bytes>'
        else:
            result[key] = value
    return result
```

## Example Log Output

```json
{"event": "request_sent", "timestamp": "2025-01-11T10:30:00Z", "raida_id": 0, "command": "Detect", "token_count": 5}
{"event": "response_received", "timestamp": "2025-01-11T10:30:00Z", "raida_id": 0, "status": 241, "status_name": "ALL_PASS", "latency_ms": 45.2}
{"event": "request_sent", "timestamp": "2025-01-11T10:30:00Z", "raida_id": 1, "command": "Detect", "token_count": 5}
{"event": "response_received", "timestamp": "2025-01-11T10:30:00Z", "raida_id": 1, "status": 241, "status_name": "ALL_PASS", "latency_ms": 52.1}
...
{"event": "consensus_result", "timestamp": "2025-01-11T10:30:01Z", "operation": "Detect", "passes": 25, "fails": 0, "consensus_achieved": true}
```

## Debug Packet Logging

For debugging only - enable with DEBUG level:

```python
def log_packet_debug(self, direction: str, raida_id: int, packet: bytes):
    if self.logger.isEnabledFor(logging.DEBUG):
        # Only log header, never body
        header_hex = packet[:32].hex()
        self.logger.debug(json.dumps({
            'event': 'packet_debug',
            'direction': direction,
            'raida_id': raida_id,
            'header_hex': header_hex,
            'total_size': len(packet)
        }))
```

## Performance Metrics

```python
class MetricsCollector:
    def __init__(self):
        self.latencies = {i: [] for i in range(25)}

    def record_latency(self, raida_id: int, latency_ms: float):
        self.latencies[raida_id].append(latency_ms)

    def get_stats(self, raida_id: int) -> dict:
        data = self.latencies[raida_id]
        if not data:
            return {}
        return {
            'min': min(data),
            'max': max(data),
            'avg': sum(data) / len(data),
            'count': len(data)
        }
```
