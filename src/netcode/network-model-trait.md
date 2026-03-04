# NetworkModel Trait & Abstractions

## The NetworkModel Trait

The netcode described above is expressed as a trait because it gives us testability, single-player support, and deployment flexibility **and** preserves architectural escape hatches. The sim and game loop never know which deployment mode is running, and they also don't need to know if deferred milestones introduce (outside the `M4` minimal-online slice):

- a compatibility bridge/protocol adapter for cross-engine experiments (e.g., community interop with legacy game versions or OpenRA)
- a replacement default netcode if production evidence reveals a serious flaw or a better architecture

The product still ships one recommended/default multiplayer path; the trait exists so changing the path under a deferred milestone does not require touching `ic-sim` or the game loop.

```rust
pub trait NetworkModel: Send + Sync {
    /// Local player submits an order
    fn submit_order(&mut self, order: TimestampedOrder);
    /// Poll for the next tick's confirmed orders (None = not ready yet)
    fn poll_tick(&mut self) -> Option<TickOrders>;
    /// Report local fast sync hash (`u64`) for desync detection
    fn report_sync_hash(&mut self, tick: u64, hash: u64);
    /// Connection/sync status
    fn status(&self) -> NetworkStatus;
    /// Diagnostic info (latency, packet loss, etc.)
    fn diagnostics(&self) -> NetworkDiagnostics;
}
```

**Trait shape note:** This trait is lockstep-shaped — `poll_tick()` returns confirmed orders or nothing, matching lockstep's "wait, then advance" pattern. All shipping implementations fit naturally. See § "Additional NetworkModel Architectures" for how this constrains non-lockstep experiments (M11).

### Deployment Modes

The same netcode runs in five modes. The first two are utility adapters (no network involved). The last three are real multiplayer deployments of the same protocol:

| Implementation         | What It Is                                        | When Used                                  | Phase   |
| ---------------------- | ------------------------------------------------- | ------------------------------------------ | ------- |
| `LocalNetwork`         | Pass-through — orders go straight to sim          | Single player, automated tests             | Phase 2 |
| `ReplayPlayback`       | File reader — feeds saved orders into sim         | Watching replays                           | Phase 2 |
| `EmbeddedRelayNetwork` | Listen server — host embeds `RelayCore` and plays | Casual, community, LAN, "Host Game" button | Phase 5 |
| `RelayLockstepNetwork` | Dedicated relay (recommended for online)          | Internet multiplayer, ranked               | Phase 5 |

`EmbeddedRelayNetwork` and `RelayLockstepNetwork` implement the same netcode. The differences are topology and trust:

- **`EmbeddedRelayNetwork`** — the host's game client runs `RelayCore` (see above) as a listen server. Other players connect to the host. Full sub-tick ordering, anti-lag-switch, and replay signing — same as a dedicated relay. The host plays normally while serving. Ideal for casual/community/LAN play: "Host Game" button, zero external infrastructure.
- **`RelayLockstepNetwork`** — clients connect to a standalone relay server on trusted infrastructure. Required for ranked/competitive play (host can't be trusted with relay authority). Recommended for internet play.

Both use adaptive run-ahead, frame resilience, delta-compressed TLV, and Ed25519 signing. They share identical `RelayCore` logic — connecting clients use `RelayLockstepNetwork` in both cases and cannot distinguish between an embedded or dedicated relay.

These deployments are the current lockstep family. The `NetworkModel` trait intentionally keeps room for deferred non-default implementations (e.g., bridge adapters, rollback experiments, fog-authoritative tournament servers) without changing sim code or invalidating the architectural boundary. Those paths are optional and not part of `M4` exit criteria.

### Example Deferred Adapter: `NetcodeBridgeModel` (Compatibility Bridge)

To make the architectural intent concrete, here is the shape of a **deferred compatibility bridge** implementation. This is not a promise of full cross-play with original RA/OpenRA; it is an example of how the `NetworkModel` boundary allows experimentation without touching `ic-sim`. Planned-deferral scope: cross-engine bridge experiments are tied to `M7.NET.CROSS_ENGINE_BRIDGE_AND_TRUST` / `M11` visual+interop follow-ons and are unranked by default unless a separate explicit decision certifies a mode.

**Use cases this enables (deferred / optional, `M7+` and `M11`):**

- Community-hosted bridge experiments for legacy game versions or OpenRA
- Discovery-layer interop plus limited live-play compatibility prototypes
- Transitional migrations if IC changes its default netcode under a separately approved deferred milestone

```rust
/// Example deferred adapter. Not part of the initial shipping set.
/// Wraps a protocol/transport bridge and translates between an external
/// protocol family and IC's canonical TickOrders interface.
pub struct NetcodeBridgeModel<B: ProtocolBridge> {
    bridge: B,
    inbound_ticks: VecDeque<TickOrders>,
    diagnostics: NetworkDiagnostics,
    status: NetworkStatus,
    // Capability negotiation / compatibility flags:
    // supported_orders, timing_model, hash_mode, etc.
}

impl<B: ProtocolBridge> NetworkModel for NetcodeBridgeModel<B> {
    fn submit_order(&mut self, order: TimestampedOrder) {
        self.bridge.submit_ic_order(order);
    }

    fn poll_tick(&mut self) -> Option<TickOrders> {
        self.bridge.poll_bridge();
        self.inbound_ticks.pop_front()
    }

    fn report_sync_hash(&mut self, tick: u64, hash: u64) {
        self.bridge.report_ic_sync_hash(tick, hash);
    }

    fn status(&self) -> NetworkStatus { self.status.clone() }
    fn diagnostics(&self) -> NetworkDiagnostics { self.diagnostics.clone() }
}
```

**What a bridge adapter is responsible for:**

- **Protocol translation** — external wire messages ↔ IC `TimestampedOrder` / `TickOrders`
- **Timing model adaptation** — map external timing/order semantics into IC tick/sub-tick expectations (or degrade gracefully with explicit fairness limits)
- **Capability negotiation** — detect unsupported features/order types and reject, stub, or map them explicitly
- **Authority/trust policy** — declare whether the bridge is relay-authoritative, P2P-trust, or observer-only
- **Diagnostics** — expose compatibility state, dropped/translated orders, and fidelity warnings via `NetworkDiagnostics`

**What a bridge adapter is NOT responsible for:**

- **Making simulations identical** across engines (D011 still applies)
- **Mutating `ic-sim` rules** to emulate foreign bugs/quirks in core engine code
- **Bypassing ranked trust rules** (bridge modes are unranked by default unless a separate explicit decision (`Dxxx` / `Pxxx`) certifies one)
- **Hiding incompatibilities** — unsupported semantics must be visible to users/operators

**Practical expectation:** Early bridge modes are most likely to ship (if ever) as **observer/replay/discovery** integrations first, then limited casual play experiments, with strict capability constraints. Competitive/ranked bridge play would require a separate explicit decision and a much stronger certification story.

**Sub-tick ordering in deferred direct-peer modes:** If a direct-peer gameplay mode is introduced later (deferred), it will not have neutral time authority. It must therefore use deterministic `(sub_tick_time, player_id)` ordering and explicitly accept reduced fairness under clock skew. This tradeoff is only defensible for explicit low-infra scenarios (for example, LAN experiments), not as the default competitive path.

### Single-Player: Zero Delay

`LocalNetwork` processes orders on the very next tick with zero scheduling delay:

```rust
impl NetworkModel for LocalNetwork {
    fn submit_order(&mut self, order: TimestampedOrder) {
        // Order goes directly into the next tick — no delay, no projection
        self.pending.push(order);
    }
    
    fn poll_tick(&mut self) -> Option<TickOrders> {
        // Accumulator-based: tracks how much real time has elapsed and
        // emits one tick per TICK_DURATION of accumulated time. This
        // preserves the fixed 30 tps rate independent of frame rate —
        // if a frame arrives late, multiple ticks are available on the
        // next calls (bounded by GameLoop's MAX_TICKS_PER_FRAME cap).
        let now = Instant::now();
        let elapsed = now - self.last_poll_time;
        if elapsed < TICK_DURATION {
            return None; // Not enough time for the next tick
        }
        // Deduct one tick's worth of time; remainder carries forward.
        // This prevents drift — late frames catch up, early frames wait.
        self.last_poll_time += TICK_DURATION;
        self.tick += 1;
        Some(TickOrders {
            tick: self.tick,
            orders: std::mem::take(&mut self.pending),
        })
    }
}
```

At 30 tps, a click-to-move in single player is confirmed within ~33 ms — imperceptible to humans (reaction time is ~200 ms). The accumulator pattern (`last_poll_time += TICK_DURATION` instead of `= now + TICK_DURATION`) ensures the sim maintains exactly 30 tps regardless of frame rate — missed time is caught up via multiple ticks per frame, bounded by `GameLoop`'s `MAX_TICKS_PER_FRAME` cap. Combined with visual prediction, the game feels **instant**.

### Replay Playback

Replays are a natural byproduct of the architecture:

```
Replay file = initial state + sequence of TickOrders
Playback = feed TickOrders through Simulation via ReplayPlayback NetworkModel
```

Replays are signed by the relay server for tamper-proofing (see `06-SECURITY.md`).

### Background Replay Writer

During live games, the replay file is written by a **background writer** using a bounded channel — the sim thread spends at most 5 ms on I/O per tick. This prevents disk write latency from causing sustained frame hitches (a problem observed in 0 A.D.'s synchronous replay recording — see `research/0ad-warzone2100-netcode-analysis.md`):

```rust
use std::sync::atomic::{AtomicU32, Ordering};
use std::time::Duration;

/// Bounded-latency replay recorder. The sim thread pushes tick frames
/// into a bounded channel; a background thread drains and writes.
pub struct BackgroundReplayWriter {
    queue: crossbeam::channel::Sender<ReplayTickFrame>,
    handle: std::thread::JoinHandle<()>,
    /// Counts frames lost due to channel backpressure (V45).
    lost_frame_count: AtomicU32,
}

impl BackgroundReplayWriter {
    /// Called from the sim thread after each tick. Blocks at most 5ms
    /// (send_timeout) — bounded latency, not lock-free.
    pub fn record_tick(&self, frame: ReplayTickFrame) {
        // crossbeam bounded channel sized for ~10 seconds of ticks
        // (300 frames at 30 tps). Use send_timeout to avoid blocking
        // the sim thread while giving the writer a chance to drain.
        // If the timeout expires, the frame is lost — tracked in the
        // replay header (see V45 mitigations below).
        match self.queue.send_timeout(frame, Duration::from_millis(5)) {
            Ok(()) => {},
            Err(_) => self.lost_frame_count.fetch_add(1, Ordering::Relaxed),
        }
    }
}
```

> **Security (V45):** The `send_timeout` pattern above bounds blocking to 5ms. If the writer still can't keep up, frames are lost — `lost_frame_count` is recorded in the replay header. Lost frames break the Ed25519 signature chain (V4). Replays with lost frames are marked `incomplete` (playable but not ranked-verifiable). The signature chain handles gaps via the `skipped_ticks` field in `TickSignature` — see `formats/save-replay-formats.md` § TickSignature and `06-SECURITY.md` § Vulnerability 45.

The background thread writes frames incrementally — the `.icrep` file is always valid (see `formats/save-replay-formats.md` § Replay File Format). If the game crashes, the replay up to the last flushed frame is recoverable. On game end, the writer flushes remaining frames, writes the final header (`total_ticks`, `final_state_hash`, `lost_frame_count`), sets the `INCOMPLETE` flag (bit 4) if any frames were lost, and closes the file.

## Additional NetworkModel Architectures

The `NetworkModel` trait enables fundamentally different networking approaches beyond the default relay lockstep. IC ships relay-assisted deterministic lockstep with sub-tick ordering as the default. Everything above this section — sub-tick, QoS calibration, timing feedback, adaptive run-ahead, never-stall relay, visual prediction — is part of that default lockstep system.

### Fog-Authoritative Server (anti-maphack)

Server runs full sim, sends each client only visible entities. Eliminates maphack architecturally. Requires server compute per game. An operator enables it per-room or per-match-type via `ic-server` capability flags (D074) — it is not a separate product. See `06-SECURITY.md` § Vulnerability 1, `decisions/09b/D074-community-server-bundle.md`, and `research/fog-authoritative-server-design.md` for the full design. Implementation milestone: `M11` (`P-Optional`), but the `NetworkModel` trait abstraction and `ic-server` capability infrastructure are designed to support it from day one.

### Rollback / GGPO-Style (`P-Optional` / `M11`)

Client predicts with local input, rolls back on misprediction. Requires snapshottable sim (D010 — already designed). Rollback and lockstep are **alternative architectures, never mixed** — none of the lockstep features above (sub-tick, calibration, timing feedback, run-ahead) would exist in a rollback `NetworkModel`. The current `NetworkModel` trait is lockstep-shaped (`poll_tick()` returns confirmed-or-nothing); rollback would need trait extension or game loop restructuring in `ic-game`. `ic-sim` stays untouched (invariant #2). Stormgate (2024) proved RTS rollback is production-viable at 64 tps; delta rollback (Dehaene, 2024) reduces snapshot cost to ~5% via Bevy `Changed<T>` change detection. Full analysis in `research/stormgate-rollback-relay-landscape-analysis.md`.

### Cross-Engine Protocol Adapter

A `ProtocolAdapter<N>` wrapper translates between Iron Curtain's native protocol and other engines' wire formats (e.g., OpenRA). Uses the `OrderCodec` trait for format translation. See `07-CROSS-ENGINE.md` for full design.

## OrderCodec: Wire Format Abstraction

For cross-engine play and protocol versioning, the wire format is abstracted behind a trait:

```rust
pub trait OrderCodec: Send + Sync {
    fn encode(&self, order: &TimestampedOrder) -> Result<Vec<u8>>;
    fn decode(&self, bytes: &[u8]) -> Result<TimestampedOrder>;
    fn protocol_id(&self) -> ProtocolId;
}

/// Native format — fast, compact, versioned (delta-compressed TLV)
pub struct NativeCodec { version: u32 }

/// Translates to/from OpenRA's wire format
pub struct OpenRACodec {
    order_map: OrderTranslationTable,
    coord_transform: CoordTransform,
}
```

See `07-CROSS-ENGINE.md` for full cross-engine compatibility design.
