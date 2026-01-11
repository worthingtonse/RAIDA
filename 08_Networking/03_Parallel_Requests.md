# Parallel Requests

## Overview

Most RAIDA operations require sending requests to all 25 RAIDA servers.
Parallel execution reduces total latency from ~75s (serial) to ~3s (parallel).

## Parallel Execution Pattern

```
Time -->
         |-- RAIDA 0 --|
         |-- RAIDA 1 --|
         |-- RAIDA 2 --|
Client --|    ...      |-- Collect Results
         |-- RAIDA 23 -|
         |-- RAIDA 24 -|
```

## Implementation Strategies

### Strategy 1: Threading (Python)

```python
import concurrent.futures

def send_to_all_raida(
    packets: list,  # 25 packets, one per RAIDA
    hosts: list,    # 25 hostnames
    timeout_ms: int = 3000
) -> list:
    results = [None] * 25

    def send_one(raida_id: int) -> tuple:
        try:
            response = send_udp_request(
                hosts[raida_id],
                raida_id,
                packets[raida_id],
                timeout_ms
            )
            return (raida_id, response, None)
        except Exception as e:
            return (raida_id, None, e)

    with concurrent.futures.ThreadPoolExecutor(max_workers=25) as executor:
        futures = [executor.submit(send_one, i) for i in range(25)]
        for future in concurrent.futures.as_completed(futures):
            raida_id, response, error = future.result()
            results[raida_id] = (response, error)

    return results
```

### Strategy 2: Async/Await (Python)

```python
import asyncio

async def send_to_all_raida_async(
    packets: list,
    hosts: list,
    timeout_ms: int = 3000
) -> list:
    async def send_one(raida_id: int) -> tuple:
        try:
            response = await asyncio.wait_for(
                send_udp_async(hosts[raida_id], raida_id, packets[raida_id]),
                timeout=timeout_ms / 1000.0
            )
            return (raida_id, response, None)
        except Exception as e:
            return (raida_id, None, e)

    tasks = [send_one(i) for i in range(25)]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results
```

### Strategy 3: Select-based Multiplexing

```python
import select
import socket

def send_to_all_raida_select(
    packets: list,
    hosts: list,
    timeout_ms: int = 3000
) -> list:
    # Create one socket per RAIDA
    sockets = {}
    for raida_id in range(25):
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        sock.setblocking(False)
        port = 18000 + raida_id
        sock.sendto(packets[raida_id], (hosts[raida_id], port))
        sockets[sock] = raida_id

    # Collect responses
    results = [None] * 25
    deadline = time.time() + (timeout_ms / 1000.0)

    while sockets and time.time() < deadline:
        remaining = deadline - time.time()
        readable, _, _ = select.select(list(sockets.keys()), [], [], remaining)

        for sock in readable:
            raida_id = sockets.pop(sock)
            try:
                response, _ = sock.recvfrom(4096)
                results[raida_id] = (response, None)
            except Exception as e:
                results[raida_id] = (None, e)
            sock.close()

    # Mark remaining as timeout
    for sock, raida_id in sockets.items():
        results[raida_id] = (None, TimeoutError("No response"))
        sock.close()

    return results
```

## Result Aggregation

After parallel execution, aggregate results for consensus:

```python
def aggregate_results(results: list, token_count: int) -> dict:
    successes = 0
    failures = 0
    errors = 0
    per_token = [[False] * 25 for _ in range(token_count)]

    for raida_id, (response, error) in enumerate(results):
        if error:
            errors += 1
            continue

        header = parse_response_header(response)
        if header.status == 241:  # ALL_PASS
            successes += 1
            for t in range(token_count):
                per_token[t][raida_id] = True
        elif header.status == 242:  # ALL_FAIL
            failures += 1
        elif header.status == 243:  # MIXED
            bitfield = parse_bitfield(response[32:-2], token_count)
            for t, passed in enumerate(bitfield):
                per_token[t][raida_id] = passed

    return {
        'successes': successes,
        'failures': failures,
        'errors': errors,
        'per_token': per_token
    }
```

## Performance Considerations

| Workers | Expected Time | Notes |
|---------|--------------|-------|
| 1 (serial) | 25 × RTT | Avoid |
| 5 | 5 × RTT | Minimal parallelism |
| 25 | 1 × RTT | Optimal |
| 50+ | 1 × RTT | Diminishing returns |

## Common Mistakes

**DON'T** process results in order received:
```python
# WRONG - Results may arrive out of order
for i, future in enumerate(futures):
    results[i] = future.result()  # Index mismatch!
```

**DO** track RAIDA ID with each result:
```python
# CORRECT - Preserve RAIDA ID mapping
raida_id, response, error = future.result()
results[raida_id] = (response, error)
```

**DON'T** fail fast on first error:
```python
# WRONG - Missing other responses
for raida in range(25):
    try:
        response = send_request(raida)
    except:
        raise  # Exits early!
```

**DO** collect all results, then analyze:
```python
# CORRECT - Collect everything
results = send_to_all_raida(packets, hosts)
successes = sum(1 for r, e in results if r and parse_status(r) == 241)
if successes >= 13:
    # Consensus achieved
```
