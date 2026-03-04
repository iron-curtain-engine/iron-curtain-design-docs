# Multi-Player Scaling (Beyond 2 Players)

The architecture supports N players with no structural changes. Every design element — deterministic lockstep, sub-tick ordering, relay server, desync detection — works for 2, 4, 8, or more players.

### How Each Component Scales

| Component             | 2 players                        | N players                        | Bottleneck                                                             |
| --------------------- | -------------------------------- | -------------------------------- | ---------------------------------------------------------------------- |
| **Lockstep sim**      | Both run identical sim           | All N run identical sim          | No change — sim processes `TickOrders` regardless of source count      |
| **Sub-tick ordering** | Sort 2 players' orders           | Sort N players' orders           | Negligible — orders per tick is small (players issue ~0-5 orders/tick) |
| **Relay server**      | Collects from 2, broadcasts to 2 | Collects from N, broadcasts to N | Linear in N. Bandwidth is tiny (orders are small)                      |
| **Desync detection**  | Compare 2 hashes                 | Compare N hashes                 | Trivial — one hash per player per tick                                 |
| **Input delay**       | Tuned to worst of 2 connections  | Tuned to worst of N connections  | **Real bottleneck** — one laggy player affects everyone                |

### Relay Topology for Multi-Player

All multiplayer uses a relay (embedded or dedicated). Both topologies are star-shaped:

```
Embedded relay (listen server — host runs RelayCore and plays)
  B → A ← C        A = host + RelayCore, full sub-tick ordering
      ↑             Host's orders go through same pipeline as everyone's
      D

Dedicated relay server (recommended for competitive)
  B → R ← C        R = standalone relay binary, trusted infrastructure
      ↑             No player has hosting advantage
      D
```

Both modes provide:
- Sub-tick ordering with neutral time authority
- Match-start QoS calibration + bounded auto-tuning
- Lag-switch protection for all players
- Replay signing

The **dedicated relay** additionally provides:
- NAT traversal for all players (no port forwarding needed)
- No player has any hosting advantage (relay is on neutral infrastructure)
- Required for ranked/competitive play (untrusted host can't manipulate relay)

The **embedded relay** (listen server) additionally provides:
- Zero external infrastructure — "Host Game" button just works
- Full `RelayCore` pipeline (no host advantage in order processing — host's orders go through sub-tick sorting like everyone else's)
- Port forwarding required (same as any self-hosted server)

### The Real Scaling Limit: Sim Cost, Not Network

With N players, the sim has more units, more orders, and more state to process. This is a **sim performance** concern, not a network concern:

- 2-player game: ~200-500 units typically
- 4-player FFA or 2v2: ~400-1000 units
- 8-player: ~800-2000 units

The performance targets in `10-PERFORMANCE.md` already account for this. The efficiency pyramid (flowfields, spatial hash, sim LOD, amortized work) is designed for 2000+ units on mid-range hardware. An 8-player game is within budget.

### Team Games (2v2, 3v3, 4v4)

Team games work identically to FFA. Each player submits orders for their own units. The sim processes all orders from all players in sub-tick chronological order. Alliances, shared vision, and team chat are sim-layer and UI-layer concerns — the network model doesn't distinguish between ally and enemy.

### Observers / Spectators

Observers receive `TickOrders` but never submit any. They run the sim locally (full state, all players' perspective). In a relay server setup, the relay can optionally delay the observer feed by N ticks to prevent live coaching.

```rust
pub struct ObserverConnection {
    pub delay_ticks: u64,        // e.g., 30 ticks (~2 seconds) for anti-coaching
    pub receive_only: bool,      // true — observer never submits orders
}
```

### Player Limits

No hard architectural limit. Practical limits:
- **Lockstep input delay** — scales with the worst connection among N players. Beyond ~8 players, the slowest player's latency dominates everyone's experience.
- **Order volume** — N players generating orders simultaneously. Still tiny bandwidth (orders are small structs, not state).
- **Sim cost** — more players = more units = more computation. The efficiency pyramid handles this up to the hardware's limit.
