# Networking Overview

## Transport Protocols

RAIDA supports two transport protocols on a single port:

| Protocol | Use Case |
|----------|----------|
| TCP | Default, Large payloads, reliability needed |
| UDP | Client can use if the network proves reliable enough and if the request and responsds use less than 1400 bytes. This is fast and light |


## Port Assignment

Each RAIDA server listens on a single port that handles both UDP and TCP:
- Port: `50000 + raida_id`
- Example: RAIDA 0 = port 50000, RAIDA 24 = port 50024

**Note:** These port assignments come from Guardian servers and can change dynamically.
Clients should obtain current endpoints via the Guardian consensus system rather than
hardcoding ports. See `05_Name_Resolution.md` for details.

## Transport Selection Decision Tree

```
START
  |
  v
Is payload > 1400 bytes?
  |
  +--YES--> Use TCP
  |
  +--NO--> Is the client able to connect to the raida without major UDP losses? (e.g., financial settlement)
              |
              +--NO--> Consider TCP with confirmation
              |
              +--YES--> Use UDP (default)
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
- `05_Name_Resolution.md` - Guardian system and endpoint discovery
- `06_Error_Handling.md` - Network error recovery
