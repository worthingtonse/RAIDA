# Command Registry

## Command Groups

| Group | Name | Description |
|-------|------|-------------|
| 0 | Status | Echo, version, diagnostics |
| 1 | Authentication | Detect, POWN |
| 2 | Healing | Tickets, fix fractured tokens |
| 8 | Locker | Put, peek, remove tokens |
| 9 | Change | Break, join denominations |

## Complete Command List

| Group | Code | Name | Document |
|-------|------|------|----------|
| 0 | 0 | Echo | `Group_0_Status/00_Echo.md` |
| 0 | 1 | Version | `Group_0_Status/01_Version.md` |
| 0 | 2 | ShowStats | `Group_0_Status/02_ShowStats.md` |
| 0 | 3 | CountCoins | `Group_0_Status/03_CountCoins.md` |
| 1 | 10 | Detect | `Group_1_Authentication/10_Detect.md` |
| 1 | 11 | DetectSum | `Group_1_Authentication/11_DetectSum.md` |
| 1 | 20 | Pown | `Group_1_Authentication/20_Pown.md` |
| 1 | 21 | PownSum | `Group_1_Authentication/21_PownSum.md` |
| 2 | 40 | GetTicket | `Group_2_Healing/40_GetTicket.md` |
| 2 | 41 | GetTicketSum | `Group_2_Healing/41_GetTicketSum.md` |
| 2 | 50 | ValidateTicket | `Group_2_Healing/50_ValidateTicket.md` |
| 2 | 60 | Find | `Group_2_Healing/60_Find.md` |
| 2 | 80 | Fix | `Group_2_Healing/80_Fix.md` |
| 4 | 40 | GetTicket | `Group_2_Healing/40_GetTicket.md` |
| 4 | 41 | GetTicketSum | `Group_2_Healing/41_GetTicketSum.md` |
| 8 | 82 | Put | `Group_8_Locker/82_Put.md` |
| 8 | 83 | Peek | `Group_8_Locker/83_Peek.md` |
| 8 | 84 | Remove | `Group_8_Locker/84_Remove.md` |
| 9 | 91 | GetAvailableSNs | `Group_9_Change/91_GetAvailableSNs.md` |
| 9 | 92 | Break | `Group_9_Change/92_Break.md` |
| 9 | 93 | Join | `Group_9_Change/93_Join.md` |

## Implementation Order

Start with these commands:
1. Echo (Group 0, Code 0) - Test connectivity
2. Detect (Group 1, Code 10) - Basic authentication
3. Pown (Group 1, Code 20) - Change ownership

## Machine-Readable Data

See `../data/commands.json` for programmatic access.
