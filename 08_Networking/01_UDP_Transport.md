# UDP Transport

## Overview

UDP is the preferred transport for RAIDA commands due to:
- Lower latency (no connection handshake)
- Simpler implementation
- Suitable for small packets (< 1400 bytes)

## Port Calculation

```python
def get_udp_port(raida_id: int) -> int:
    return 18000 + raida_id
```

## Maximum Packet Size

| Constraint | Value |
|------------|-------|
| UDP max theoretical | 65535 bytes |
| Ethernet MTU | 1500 bytes |
| Safe payload size | 1400 bytes |
| RAIDA recommended max | 1400 bytes |

## Request/Response Pattern

UDP uses a simple request/response model:

```
Client                          RAIDA Server
   |                                 |
   |-------- Request Packet -------->|
   |                                 |
   |<------- Response Packet --------|
   |                                 |
```

## Socket Configuration

```python
import socket

def create_udp_socket(timeout_ms: int = 3000) -> socket.socket:
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(timeout_ms / 1000.0)
    return sock
```

## Sending a Request

```python
def send_udp_request(
    host: str,
    raida_id: int,
    packet: bytes,
    timeout_ms: int = 3000
) -> bytes:
    port = 18000 + raida_id
    sock = create_udp_socket(timeout_ms)

    try:
        sock.sendto(packet, (host, port))
        response, addr = sock.recvfrom(4096)
        return response
    except socket.timeout:
        raise TimeoutError(f"RAIDA {raida_id} did not respond")
    finally:
        sock.close()
```

## Retry Strategy

```python
def send_with_retry(
    host: str,
    raida_id: int,
    packet: bytes,
    timeout_ms: int = 3000,
    max_retries: int = 2
) -> bytes:
    last_error = None

    for attempt in range(max_retries + 1):
        try:
            return send_udp_request(host, raida_id, packet, timeout_ms)
        except TimeoutError as e:
            last_error = e
            # Increase timeout on retry
            timeout_ms = int(timeout_ms * 1.5)

    raise last_error
```

## Common Mistakes

**DON'T** create a new socket per RAIDA in parallel requests:
```python
# WRONG - May exhaust file descriptors
results = []
for raida in range(25):
    sock = socket.socket(...)  # 25 sockets!
    results.append(send_async(sock, raida, packet))
```

**DO** reuse sockets or use proper pooling:
```python
# CORRECT - Single socket, sequential
sock = create_udp_socket()
for raida in range(25):
    sock.sendto(packet, (host, 18000 + raida))
    # ... collect responses
sock.close()
```

**DON'T** use blocking receives in parallel:
```python
# WRONG - Will block on first RAIDA
for raida in range(25):
    response = sock.recvfrom(4096)  # Blocks!
```

**DO** use select() or async for parallel:
```python
# CORRECT - Use select for multiplexing
import select
readable, _, _ = select.select([sock], [], [], timeout)
if readable:
    response, addr = sock.recvfrom(4096)
```
