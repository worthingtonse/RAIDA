# System Architecture

## RAIDA Network

The RAIDA consists of 25 independent servers (RAIDA 0-24) distributed
geographically. Each server:

- Maintains its own copy of the token database
- Operates independently (no inter-server consensus required for auth)
- Stores one AN per token (different AN on each RAIDA)

## Consensus Model

- **Authentic**: Token passes on ≥13 RAIDA
- **Counterfeit**: Token fails on ≥13 RAIDA
- **Fractured**: Mixed results (requires healing)

## Client-Server Flow

```
Client                          RAIDA 0-24
  |                                 |
  |--[Request to all 25 RAIDA]----->|
  |                                 |
  |<--[25 parallel responses]-------|
  |                                 |
  [Aggregate results locally]
```

## Token Structure

Each token has:
- 1-byte denomination code
- 4-byte serial number
- 25 x 16-byte authenticity numbers (one per RAIDA)

Total: 405 bytes per token (5 + 400)

## Security Model

- No central authority
- No single point of failure
- Each RAIDA admin cannot know other RAIDA's ANs
- 13-of-25 threshold prevents minority attacks
