# 03 — Network Architecture

## Our Netcode

Iron Curtain ships **one default gameplay netcode** today: relay-assisted deterministic lockstep with sub-tick order fairness. This is the recommended production path, not a buffet of equal options in the normal player UX. The `NetworkModel` trait still exists for more than testing: it lets us run single-player and replay modes cleanly, support multiple deployments (dedicated relay / embedded relay), and preserve the ability to introduce deferred compatibility bridges or replace the default netcode under explicitly deferred milestones (for example `M7+` interop experiments or `M11` optional architecture work) if evidence warrants it (e.g., cross-engine interop experiments, architectural flaws discovered in production). Those paths require explicit decision/tracker placement and are not part of `M4` exit criteria.

Scope note: in this chapter, "P2P" refers only to direct gameplay transport (a deferred/optional mode), not Workshop/content distribution. Workshop P2P remains in scope via D049/D074.

**Keywords:** netcode, relay lockstep, `NetworkModel`, sub-tick timestamps, reconnection, desync debugging, replay determinism, compatibility bridge, ranked authority, relay server

Key influences:
- **Counter-Strike 2** — sub-tick timestamps for order fairness
- **C&C Generals/Zero Hour** — adaptive run-ahead, frame resilience, delta-compressed wire format, disconnect handling
- **Valve GameNetworkingSockets (GNS)** — ack vector reliability, message lanes with priority/weight, per-ack RTT measurement, pluggable signaling, transport encryption, Nagle-style batching (see `research/valve-github-analysis.md`)
- **OpenTTD** — multi-level desync debugging, token-based liveness, reconnection via state transfer
- **Minetest** — time-budget rate control (LagPool), half-open connection defense
- **OpenRA** — what to avoid: TCP stalling, static order latency, shallow sync buffers
- **Bryant & Saiedian (2021)** — state saturation taxonomy, traffic class segregation

## The Protocol

All protocol types live in the `ic-protocol` crate — the ONLY shared dependency between sim and net:

```rust
#[derive(Clone, Serialize, Deserialize, Hash)]
pub enum PlayerOrder {
    Move { unit_ids: Vec<UnitId>, target: WorldPos },
    Attack { unit_ids: Vec<UnitId>, target: Target },
    Build { structure: StructureType, position: WorldPos },
    SetRallyPoint { building: BuildingId, position: WorldPos },
    Sell { building: BuildingId },
    Idle,  // Explicit no-op — keeps player in the tick's order list for timing/presence
    // ... every possible player action
}

/// Sub-tick timestamp on every order (CS2-inspired, see below).
/// In relay modes this is a client-submitted timing hint that the relay
/// normalizes/clamps before broadcasting canonical TickOrders.
#[derive(Clone, Serialize, Deserialize)]
pub struct TimestampedOrder {
    pub player: PlayerId,
    pub order: PlayerOrder,
    pub sub_tick_time: u32,  // microseconds within the tick window (0 = tick start)
}
// NOTE: sub_tick_time is an integer (microseconds offset from tick start).
// At 15 ticks/sec the tick window is ~66,667µs — u32 is more than sufficient.
// Integer ordering avoids any platform-dependent float comparison behavior
// and keeps ic-protocol free of floating-point types entirely.

pub struct TickOrders {
    pub tick: u64,
    pub orders: Vec<TimestampedOrder>,
}

impl TickOrders {
    /// CS2-style: process in chronological order within the tick.
    /// Uses a caller-provided scratch buffer to avoid per-tick heap allocation.
    /// The buffer is cleared and reused each tick (see TickScratch pattern in 10-PERFORMANCE.md).
    /// Tie-break by player ID so equal timestamps remain deterministic if a
    /// deferred non-relay mode is ever enabled. Relay modes already emit
    /// canonical normalized timestamps, but the helper remains safe.
    pub fn chronological<'a>(&'a self, scratch: &'a mut Vec<&'a TimestampedOrder>) -> &'a [&'a TimestampedOrder] {
        scratch.clear();
        scratch.extend(self.orders.iter());
        scratch.sort_by_key(|o| (o.sub_tick_time, o.player));
        scratch.as_slice()
    }
}
```
