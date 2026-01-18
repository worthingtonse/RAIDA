# Timeout Strategy

## Overview

Proper timeout configuration balances:
- **Speed**: Don't wait too long for slow/dead servers
- **Reliability**: Don't give up too quickly on busy servers

## Default Timeouts

| Operation | Initial | After Retry | Maximum |
|-----------|---------|-------------|---------|
| UDP Echo | 2000 ms | 3000 ms | 5000 ms |
| UDP Detect | 3000 ms | 4500 ms | 6000 ms |
| UDP POWN | 3000 ms | 4500 ms | 6000 ms |
| TCP Connect | 5000 ms | N/A | 5000 ms |
| TCP Read | 10000 ms | N/A | 10000 ms |

## Timeout Calculation

```python
def calculate_timeout(
    base_timeout_ms: int,
    token_count: int,
    attempt: int
) -> int:
    # Add time for large batches
    batch_factor = 1 + (token_count / 100)

    # Increase on retry
    retry_factor = 1.5 ** attempt

    timeout = base_timeout_ms * batch_factor * retry_factor

    # Cap at maximum
    return min(int(timeout), 10000)
```

## Retry Strategy

```python
def send_with_backoff(
    raida_id: int,
    packet: bytes,
    base_timeout_ms: int = 3000,
    max_retries: int = 2
) -> tuple:
    """
    Returns: (response, attempts_made, final_timeout)
    """
    for attempt in range(max_retries + 1):
        timeout = calculate_timeout(base_timeout_ms, 0, attempt)

        try:
            response = send_udp_request(raida_id, packet, timeout)
            return (response, attempt + 1, timeout)
        except TimeoutError:
            if attempt == max_retries:
                raise
            continue
```

## Adaptive Timeouts

Track response times and adjust:

```python
class AdaptiveTimeout:
    def __init__(self, base_ms: int = 3000):
        self.base_ms = base_ms
        self.history = [base_ms] * 10  # Rolling window
        self.index = 0

    def record(self, response_time_ms: int):
        self.history[self.index] = response_time_ms
        self.index = (self.index + 1) % len(self.history)

    def get_timeout(self) -> int:
        avg = sum(self.history) / len(self.history)
        # 2x average + buffer
        return int(max(avg * 2 + 500, self.base_ms))
```

## Per-RAIDA Timeout Tracking

```python
class RAIDATimeoutManager:
    def __init__(self):
        self.timeouts = [AdaptiveTimeout() for _ in range(25)]
        self.failures = [0] * 25

    def get_timeout(self, raida_id: int) -> int:
        base = self.timeouts[raida_id].get_timeout()
        # Increase for unreliable RAIDA
        if self.failures[raida_id] > 3:
            base = int(base * 1.5)
        return min(base, 10000)

    def record_success(self, raida_id: int, response_time_ms: int):
        self.timeouts[raida_id].record(response_time_ms)
        self.failures[raida_id] = max(0, self.failures[raida_id] - 1)

    def record_failure(self, raida_id: int):
        self.failures[raida_id] += 1
```

## Consensus-Aware Early Termination

Don't wait for all 25 when consensus is achieved:

```python
async def send_until_consensus(
    packets: list,
    hosts: list,
    required: int = 13,
    timeout_ms: int = 5000
) -> list:
    results = [None] * 25
    successes = 0
    completed = 0

    async def send_and_track(raida_id: int):
        nonlocal successes, completed
        try:
            response = await send_async(raida_id, packets[raida_id])
            results[raida_id] = response
            if is_success(response):
                successes += 1
        finally:
            completed += 1

    tasks = [asyncio.create_task(send_and_track(i)) for i in range(25)]

    # Wait until consensus or all complete
    while successes < required and completed < 25:
        await asyncio.sleep(0.01)

    # Cancel remaining if consensus achieved
    if successes >= required:
        for task in tasks:
            if not task.done():
                task.cancel()

    return results
```

## Common Mistakes

**DON'T** use same timeout for all operations:
```python
# WRONG
TIMEOUT = 3000
send_echo(timeout=TIMEOUT)
send_pown_100_tokens(timeout=TIMEOUT)  # Too short!
```

**DO** scale timeout with payload:
```python
# CORRECT
echo_timeout = 2000
pown_timeout = 3000 + (token_count * 50)
```

**DON'T** retry indefinitely:
```python
# WRONG
while True:
    try:
        return send_request()
    except TimeoutError:
        continue  # Infinite loop!
```

**DO** cap retries and fail gracefully:
```python
# CORRECT
for attempt in range(3):
    try:
        return send_request()
    except TimeoutError:
        if attempt == 2:
            raise
```
