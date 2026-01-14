# Name Resolution

## Overview

RAIDA clients discover server endpoints through a consensus-based Guardian system
rather than traditional DNS. This provides:

- **DNS independence**: Only Guardian FQDNs require DNS resolution
- **Fault tolerance**: Multiple Guardians provide redundancy
- **Integrity**: Consensus prevents manipulation by compromised Guardians
- **Dynamic updates**: Endpoints can change without client updates

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT                                   │
├─────────────────────────────────────────────────────────────────┤
│  1. Check local hints (if enabled)                              │
│  2. Check cache file (if fresh)                                 │
│  3. Query ALL Guardians concurrently                            │
│  4. Validate consensus (3+ must agree)                          │
│  5. Cache results for 72 hours                                  │
└───────────────────────┬─────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │Guardian │    │Guardian │    │Guardian │  ... (26 total)
   │   #1    │    │   #2    │    │   #3    │
   └────┬────┘    └────┬────┘    └────┬────┘
        │              │              │
        └──────────────┼──────────────┘
                       │ Returns: coin{N}.txt
                       ▼
              ┌─────────────────┐
              │   Host File     │
              │ 192.168.1.1:50000 0-0-0-P │
              │ 192.168.1.2:50001 1-0-0-P │
              │ ...             │
              └─────────────────┘
```

## Data Sources (Priority Order)

### 1. Local RAIDA Hints (Highest Priority)

For testing or private networks, bypass Guardians entirely:

```toml
# config.toml
[main]
use_local_raidas = true
```

When enabled, the client uses hardcoded `LocalRaidas[]` directly.

### 2. Cache File

- **Location**: `._id{coin_id}.guardians` in app root
- **Format**: 25 primary lines, blank line, 25 backup lines
- **Expiration**: 72 hours (3 days)

```
192.168.1.10:50000
192.168.1.11:50001
...
192.168.1.34:50024

192.168.1.110:50000
192.168.1.111:50001
...
192.168.1.134:50024
```

### 3. Guardian Servers

26 Guardian FQDNs are hardcoded in client software:

| # | Guardian FQDN |
|---|---------------|
| 1 | raida-guardian-tx.us |
| 2 | g2.cloudcoin.asia |
| 3 | guardian.ladyjade.cc |
| 4 | watchdog.guardwatch.cc |
| 5 | g5.raida-guardian.us |
| 6 | goodguardian.xyz |
| 7 | g7.ccraida.com |
| 8 | raidaguardian.nz |
| 9 | g9.guardian9.net |
| 10 | g10.guardian25.com |
| 11 | g11.raidacash.com |
| 12 | g12.aeroflightcb300.com |
| 13 | g13.stomarket.co |
| 14 | guardian14.gsxcover.com |
| 15 | guardian.keilagd.cc |
| 16 | g16.guardianstl.us |
| 17 | raida-guardian.net |
| 18 | g18.raidaguardian.al |
| 19 | g19.paolobui.com |
| 20 | g20.cloudcoins.asia |
| 21 | guardian21.guardian.al |
| 22 | rg.encrypting.us |
| 23 | g23.cuvar.net |
| 24 | guardian24.rsxcover.com |
| 25 | g25.mattyd.click |
| 26 | g26.cloudcoinconsortium.art |

## Client Initialization Flow

```python
def initialize_raida_endpoints(coin_id: int) -> tuple[list, list]:
    """
    Returns (primary_endpoints, backup_endpoints) for the given coin.
    Each endpoint is "host:port" string.
    """

    # Step 1: Check local hints override
    if config.use_local_raidas:
        return (config.local_raidas, config.local_raidas)

    # Step 2: Check cache
    cache_file = f"._id{coin_id}.guardians"
    if cache_is_valid(cache_file, max_age_hours=72):
        return load_from_cache(cache_file)

    # Step 3: Query all Guardians concurrently
    guardian_responses = query_all_guardians(coin_id)

    # Step 4: Validate consensus
    primary, backup = find_consensus(guardian_responses, min_agreement=3)
    if primary is None:
        raise Error("Failed to reach Guardian consensus")

    # Step 5: Cache results
    save_to_cache(cache_file, primary, backup)

    return (primary, backup)
```

## Guardian Query

```python
def query_guardian(guardian_fqdn: str, coin_id: int) -> dict:
    """
    Fetch host file from a single Guardian.
    Returns dict with 'primary' and 'backup' endpoint lists.
    """
    url = f"https://{guardian_fqdn}/coin{coin_id}.txt"

    # 3 second timeout, TLS verification disabled
    response = http_get(url, timeout_ms=3000, verify_tls=False)

    primary = [None] * 25
    backup = [None] * 25

    for line in response.text.split('\n'):
        # Format: <host>:<port> <raida_idx>-<x>-<x>-<P|M>
        match = parse_line(line)
        if match:
            host, port, idx, type_flag = match
            endpoint = f"{host}:{port}"

            if type_flag == 'P':
                primary[idx] = endpoint
            elif type_flag == 'M':
                backup[idx] = endpoint

    return {'primary': primary, 'backup': backup}
```

## Host File Format

Each Guardian serves `coin{N}.txt` files over HTTPS:

```
# Format: <host>:<port> <raida_index>-<unused>-<unused>-<type>
# Type: P = Primary, M = Mirror/Backup

192.168.1.10:50000 0-0-0-P
192.168.1.11:50001 1-0-0-P
192.168.1.12:50002 2-0-0-P
...
192.168.1.34:50024 24-0-0-P
192.168.1.110:50000 0-0-0-M
192.168.1.111:50001 1-0-0-M
...
192.168.1.134:50024 24-0-0-M
```

**Field definitions:**

| Field | Description |
|-------|-------------|
| host | IPv4 address or resolvable hostname |
| port | Integer port (typically 50000 + raida_index) |
| raida_index | 0-24, identifies which RAIDA server |
| type | P = primary endpoint, M = backup/mirror |

## Consensus Validation

```python
def find_consensus(responses: list[dict], min_agreement: int = 3) -> tuple:
    """
    Find endpoint list agreed upon by at least min_agreement Guardians.
    Uses hash comparison for efficiency.
    """
    hash_counts = {}
    hash_to_data = {}

    for response in responses:
        if response is None:
            continue

        # Concatenate all endpoints and hash
        data_str = ''.join(response['primary'] + response['backup'])
        data_hash = sha256(data_str)

        hash_counts[data_hash] = hash_counts.get(data_hash, 0) + 1
        hash_to_data[data_hash] = response

        # Check if we have consensus
        if hash_counts[data_hash] >= min_agreement:
            return (response['primary'], response['backup'])

    return (None, None)  # No consensus reached
```

## RAIDA Server Peer Discovery

RAIDA servers do NOT use the Guardian system. Instead, they read peer
addresses from a static configuration file:

```toml
# config.toml on each RAIDA server
[server]
raida_id = 14
coin_id = 6
port = 50014
raida_servers = [
  "raida0.example.com:50000",
  "raida1.example.com:50001",
  "raida2.example.com:50002",
  # ... all 25 servers
]
```

At startup, each server:
1. Parses the `raida_servers` array
2. Resolves each hostname via DNS (`getaddrinfo`)
3. Stores resolved IP addresses for peer communication

## Security Properties

| Property | Mechanism |
|----------|-----------|
| Integrity | 3+ Guardians must agree (hash consensus) |
| Availability | 26 Guardians, only 3 needed |
| Confidentiality | TLS encryption (certificates not validated) |
| DNS Independence | Only Guardian FQDNs need DNS; RAIDA endpoints work directly |

**Why TLS certificates aren't validated:**
- Integrity comes from multi-Guardian consensus, not certificate chains
- Allows Guardians to use self-signed certificates
- Reduces centralized dependencies (no CA required)

## Future: Distributed Resource Directory

The Guardian system is scheduled to be replaced by the Distributed Resource
Directory (DRD), which will provide fully decentralized endpoint discovery.
Until DRD is implemented, the Guardian consensus system remains authoritative.

## Related Documents

- `00_Overview.md` - Transport protocol selection
- `01_UDP_Transport.md` - UDP communication details
- `02_TCP_Transport.md` - TCP communication details
