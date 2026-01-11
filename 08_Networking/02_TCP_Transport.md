# TCP Transport

## Overview

TCP is used when:
- Payload exceeds 1400 bytes
- Reliability is critical
- Ordered delivery is required

## Port Calculation

```python
def get_tcp_port(raida_id: int) -> int:
    return 19000 + raida_id
```

## Connection Pattern

```
Client                          RAIDA Server
   |                                 |
   |-------- SYN ---------------->   |
   |<------- SYN-ACK ------------|   |
   |-------- ACK ---------------->   |
   |                                 |
   |-------- Request Data -------->  |
   |<------- Response Data --------|  |
   |                                 |
   |-------- FIN ---------------->   |
   |<------- FIN-ACK ------------|   |
```

## Socket Configuration

```python
import socket

def create_tcp_socket(
    host: str,
    raida_id: int,
    connect_timeout_ms: int = 5000
) -> socket.socket:
    port = 19000 + raida_id
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(connect_timeout_ms / 1000.0)
    sock.connect((host, port))
    return sock
```

## Sending a Request

```python
def send_tcp_request(
    host: str,
    raida_id: int,
    packet: bytes,
    connect_timeout_ms: int = 5000,
    read_timeout_ms: int = 10000
) -> bytes:
    sock = create_tcp_socket(host, raida_id, connect_timeout_ms)

    try:
        # Send request
        sock.sendall(packet)

        # Read response header first (32 bytes)
        sock.settimeout(read_timeout_ms / 1000.0)
        header = recv_exact(sock, 32)

        # Parse body size from header (bytes 9-11)
        body_size = int.from_bytes(header[9:12], 'big')

        # Read body if present
        if body_size > 0:
            body = recv_exact(sock, body_size + 2)  # +2 for terminator
            return header + body
        else:
            return header

    finally:
        sock.close()
```

## Reading Exact Bytes

```python
def recv_exact(sock: socket.socket, n: int) -> bytes:
    """Read exactly n bytes from socket."""
    data = b''
    while len(data) < n:
        chunk = sock.recv(n - len(data))
        if not chunk:
            raise ConnectionError("Connection closed")
        data += chunk
    return data
```

## When to Use TCP

| Scenario | Transport |
|----------|-----------|
| Echo command | UDP |
| Detect 1-50 tokens | UDP |
| Detect 50+ tokens | TCP |
| POWN 1-30 tokens | UDP |
| POWN 30+ tokens | TCP |
| Batch operations | TCP |
| Locker operations (large) | TCP |

## Token Count Thresholds

Calculate when to switch to TCP:

```python
def should_use_tcp(command_code: int, token_count: int) -> bool:
    # Calculate body size
    if command_code == 10:  # Detect
        body_size = 16 + (21 * token_count) + 2
    elif command_code == 20:  # POWN
        body_size = 16 + (37 * token_count) + 2
    else:
        body_size = 0

    total_size = 32 + body_size  # header + body
    return total_size > 1400
```

## Common Mistakes

**DON'T** leave connections open:
```python
# WRONG - Connection leak
sock = create_tcp_socket(host, raida_id)
sock.sendall(packet)
response = sock.recv(4096)
# Missing sock.close()!
```

**DO** use try/finally or context managers:
```python
# CORRECT
sock = create_tcp_socket(host, raida_id)
try:
    sock.sendall(packet)
    response = sock.recv(4096)
finally:
    sock.close()
```

**DON'T** assume recv() returns all data:
```python
# WRONG - May get partial response
response = sock.recv(4096)
```

**DO** read until complete:
```python
# CORRECT - Read exact header, then body
header = recv_exact(sock, 32)
body_size = parse_body_size(header)
body = recv_exact(sock, body_size)
```
