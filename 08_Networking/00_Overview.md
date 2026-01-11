# Networking Overview

## Transport Protocols

RAIDA supports two transport protocols:

| Protocol | Port | Use Case |
|----------|------|----------|
| UDP | 18000-18024 | Default, fast, small packets |
| TCP | 19000-19024 | Large payloads, reliability needed |

## Port Assignment

Each RAIDA server has dedicated ports:
- UDP: `18000 + raida_id` (e.g., RAIDA 0 = port 18000)
- TCP: `19000 + raida_id` (e.g., RAIDA 0 = port 19000)

## Transport Selection Decision Tree

```
START
  |
  v
Is payload > 1400 bytes?
  |
  +--YES--> Use TCP
  |
  +--NO--> Is reliability critical? (e.g., financial settlement)
              |
              +--YES--> Consider TCP with confirmation
              |
              +--NO--> Use UDP (default)
```

## Default Timeouts

| Operation | Timeout | Retries |
|-----------|---------|---------|
| UDP single request | 3000 ms | 2 |
| UDP batch request | 5000 ms | 1 |
| TCP connect | 5000 ms | 1 |
| TCP request | 10000 ms | 0 |

## Related Documents

- `01_UDP_Transport.md` - UDP-specific details
- `02_TCP_Transport.md` - TCP-specific details
- `03_Parallel_Requests.md` - Sending to all 25 RAIDA
- `04_Timeout_Strategy.md` - Timeout calculation
- `05_Error_Handling.md` - Network error recovery
