# Wire Format & Message Lanes

### Frame Data Resilience (from C&C Generals + Valve GNS)

UDP is unreliable — packets can arrive corrupted, duplicated, reordered, or not at all. Inspired by C&C Generals' `FrameDataManager` (see `research/generals-zero-hour-netcode-analysis.md`), our frame data handling uses a three-state readiness model rather than a simple ready/waiting binary:

```rust
pub enum FrameReadiness {
    Ready,                     // All orders received and verified
    Waiting,                   // Still expecting orders from one or more players
    Corrupted { from: PlayerId }, // Orders received but failed integrity check — request resend
}
```

When `Corrupted` is detected, the system automatically requests retransmission from the specific player (or relay). A circular buffer retains the last N ticks of sent frame data (Generals used 65 frames) so resend requests can be fulfilled without re-generating the data.

This is strictly better than pure "missed deadline → Idle" fallback: a corrupted packet that arrives on time gets a second chance via resend rather than being silently replaced with no-op. The deadline-based Idle fallback remains as the last resort if resend also fails.

#### Ack Vector Reliability Model (from Valve GNS)

The reliability layer uses **ack vectors** — a compact bitmask encoding which of the last N packets were received — rather than TCP-style cumulative acknowledgment or selective ACK (SACK). This approach is borrowed from Valve's GameNetworkingSockets (which in turn draws from DCCP, RFC 4340). See `research/valve-github-analysis.md` § Part 1.

**How it works:** Every outgoing packet includes an ack vector — a bitmask where each bit represents a recently received packet from the peer. Bit 0 = the most recently received packet (identified by its sequence number in the header), bit 1 = the one before that, etc. A 64-bit ack vector covers the last 64 packets. The sender inspects incoming ack vectors to determine which of its sent packets were received and which were lost.

```rust
/// Included in every outgoing packet. Tells the peer which of their
/// recent packets we received.
pub struct AckVector {
    /// Sequence number of the most recently received packet (bit 0).
    pub latest_recv_seq: u32,
    /// Bitmask: bit N = 1 means we received (latest_recv_seq - N).
    /// 64 bits covers the last 64 packets at 30 tps ≈ ~2 seconds of history.
    pub received_mask: u64,
}
```

**Why ack vectors over TCP-style cumulative ACKs:**
- **No head-of-line blocking.** TCP's cumulative ACK stalls retransmission decisions when a single early packet is lost but later packets arrive fine. Ack vectors give per-packet reception status instantly.
- **Sender-side retransmit decisions.** The sender has full information about which packets were received and decides what to retransmit. The receiver never requests retransmission — it simply reports what it got. This keeps the receiver stateless with respect to reliability.
- **Natural fit for UDP.** Ack vectors assume an unreliable, unordered transport — exactly what UDP provides. On reliable transports (WebSocket), the ack vector still works but retransmit timers never fire (same "always run reliability" principle from D054).
- **Compact.** A 64-bit bitmask + 4-byte sequence number = 12 bytes per packet. TCP's SACK option can be up to 40 bytes.

**Retransmission:** When the sender sees a gap in the ack vector (bit = 0 for a packet older than the latest ACK'd), it schedules retransmission. Retransmission uses exponential backoff per packet. The retransmit buffer is the same circular buffer used for frame resilience (last N ticks of sent data).

#### Per-Ack RTT Measurement (from Valve GNS)

Each outgoing packet embeds a small **delay field** — the time elapsed between receiving the peer's most recent packet and sending this response. The peer subtracts this processing delay from the observed round-trip to compute a precise one-way latency estimate:

```rust
/// Embedded in every packet header alongside the ack vector.
pub struct PeerDelay {
    /// Microseconds between receiving the peer's latest packet
    /// and sending this packet. The peer uses this to compute RTT:
    /// RTT = (time_since_we_sent_the_acked_packet) - peer_delay
    pub delay_us: u16,
}
```

**Why this matters:** Traditional RTT measurement requires dedicated ping/pong packets or timestamps that consume bandwidth. By embedding delay in every ack, RTT is measured continuously on every packet exchange — no separate ping packets needed. This provides smoother, more accurate latency data for adaptive run-ahead (see above) and removes the ~50ms ping interval overhead. The technique is standard in Valve's GNS and is also used by QUIC (RFC 9000).

#### Nagle-Style Order Batching (from Valve GNS)

Player orders are not sent immediately on input — they are batched within each tick window and flushed at tick boundaries:

```rust
/// Order batching within a tick window.
/// Orders accumulate in a buffer and are flushed as a single packet
/// at the tick boundary. This reduces packet count by ~5-10x during
/// burst input (selecting and commanding multiple groups rapidly).
pub struct OrderBatcher {
    /// Orders accumulated since last flush.
    pending: Vec<TimestampedOrder>,
    /// Flush when the tick boundary arrives (external trigger from game loop).
    /// Unlike TCP Nagle (which flushes on ACK), we flush on a fixed cadence
    /// aligned to the sim tick rate — deterministic, predictable latency.
    tick_rate: Duration,
}
```

Unlike TCP's Nagle algorithm (which flushes on receiving an ACK — coupling send timing to network conditions), IC flushes on a fixed tick cadence. This gives deterministic, predictable send timing: all orders within a tick window are batched into one packet, sent at the tick boundary. At 30 tps, this means at most ~33ms of batching delay — well within the adaptive run-ahead window and invisible to the player. The technique is validated by Valve's GNS batching strategy (see `research/valve-github-analysis.md` § 1.7).

### Wire Format: Delta-Compressed TLV (from C&C Generals)

> **Wire protocol status:** `research/relay-wire-protocol-design.md` is the detailed protocol design draft. Normative policy bounds/defaults live in this chapter and `decisions/09b/D060-netcode-params.md`. If there is any mismatch, the decision docs are authoritative until the protocol draft is refreshed.

Inspired by C&C Generals' `NetPacket` format (see `research/generals-zero-hour-netcode-analysis.md`), the native wire format uses delta-compressed tag-length-value (TLV) encoding:

- **Tag bytes** — single ASCII byte identifies the field: `T`ype, `K`(tic**K**), `P`layer, `S`ub-tick, `D`ata
- **Delta encoding** — fields are only written when they differ from the previous order in the same packet. If the same player sends 5 orders on the same tick, the player ID and tick number are written once.
- **Empty-tick compression** — ticks with no orders compress to a single byte (Generals used `Z`). In a typical RTS, ~80% of ticks have zero orders from any given player.
- **Varint encoding** — integer fields use variable-length encoding (LEB128) where applicable. Small values (tick deltas, player indices) compress to 1-2 bytes instead of fixed 4-8 bytes. Integers that are typically small (order counts, sub-tick offsets) benefit most; fixed-size fields (hashes, signatures) remain fixed.
- **MTU-aware packet sizing** — packets stay under 476 bytes (single IP fragment, no UDP fragmentation). Fragmented UDP packets multiply loss probability — if any fragment is lost, the entire packet is dropped.
- **Transport-agnostic framing** — the wire format is independent of the underlying transport (UDP, WebSocket, QUIC). The same TLV encoding works on all transports; only the packet delivery mechanism changes (D054). This follows GNS's approach of transport-agnostic SNP (Steam Networking Protocol) frames (see `research/valve-github-analysis.md` § Part 1).

For typical RTS traffic (0-2 orders per player per tick, long stretches of idle), this compresses wire traffic by roughly 5-10x compared to naively serializing every `TimestampedOrder`.

For cross-engine play, the wire format is abstracted behind an `OrderCodec` trait — see `07-CROSS-ENGINE.md`.

### Message Lanes (from Valve GNS)

Not all network messages have equal priority. Valve's GNS introduces **lanes** — independent logical streams within a single connection, each with configurable priority and weight. IC adopts this concept for its relay protocol to prevent low-priority traffic from delaying time-critical orders.

```rust
/// Message lanes — independent priority streams within a Transport connection.
/// Each lane has its own send queue. The transport drains queues by priority
/// (higher first) and weight (proportional bandwidth among same-priority lanes).
///
/// Lanes are a `NetworkModel` concern, not a `Transport` concern — Transport
/// provides a single byte pipe; NetworkModel multiplexes lanes over it.
/// This keeps Transport implementations simple (D054).
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub enum MessageLane {
    /// Tick orders — highest priority, real-time critical.
    /// Delayed orders cause Idle substitution (anti-lag-switch).
    Orders = 0,
    /// Sync hashes, ack vectors, RTT measurements — protocol control.
    /// Must arrive promptly for desync detection and adaptive run-ahead.
    Control = 1,
    /// Chat messages, player status updates, lobby state.
    /// Important but not time-critical — can tolerate ~100ms extra delay.
    Chat = 2,
    /// Voice-over-IP frames (Opus-encoded). Real-time but best-effort —
    /// dropped frames use Opus PLC, not retransmit. See D059.
    Voice = 3,
    /// Replay data, observer feeds, telemetry.
    /// Lowest priority — uses spare bandwidth only.
    Bulk = 4,
}

/// Lane configuration — priority and weight determine scheduling.
pub struct LaneConfig {
    /// Higher priority lanes are drained first (0 = highest).
    pub priority: u8,
    /// Weight for proportional bandwidth sharing among same-priority lanes.
    /// E.g., two lanes at priority 1 with weights 3 and 1 get 75%/25% of
    /// remaining bandwidth after higher-priority lanes are satisfied.
    pub weight: u8,
    /// Per-lane buffering limit (bytes). If exceeded, oldest messages
    /// in the lane are dropped (unreliable lanes) or the lane stalls
    /// (reliable lanes). Prevents low-priority bulk data from consuming
    /// unbounded memory.
    pub buffer_limit: usize,
}
```

**Default lane configuration:**

| Lane      | Priority | Weight | Buffer | Reliability | Rationale                                               |
| --------- | -------- | ------ | ------ | ----------- | ------------------------------------------------------- |
| `Orders`  | 0        | 1      | 4 KB   | Reliable    | Orders must arrive; missed = Idle (deadline is the cap) |
| `Control` | 0        | 1      | 2 KB   | Unreliable  | Latest sync hash wins; stale hashes are useless         |
| `Chat`    | 1        | 1      | 8 KB   | Reliable    | Chat messages should arrive but can wait                |
| `Voice`   | 1        | 2      | 16 KB  | Unreliable  | Real-time voice; dropped frames use Opus PLC (D059)     |
| `Bulk`    | 2        | 1      | 64 KB  | Unreliable  | Telemetry/observer data uses spare bandwidth            |

The Orders and Control lanes share the highest priority tier — both are drained before any Chat or Bulk data is sent. Chat and Voice share priority tier 1 with a 2:1 weight ratio (voice gets more bandwidth because it's time-sensitive). This ensures that a player spamming chat messages, voice traffic, or a spectator feed generating bulk data never delays order delivery. The lane system is optional for `LocalNetwork` and `MemoryTransport` (where bandwidth is unlimited), but critical for the relay deployment where bandwidth to each client is finite. See `decisions/09g/D059-communication.md` for the full VoIP architecture.

**Relay server poll groups:** In a relay deployment serving multiple concurrent games, each game session's connections are grouped into a **poll group** (terminology from GNS). The relay's event loop polls all connections within a poll group together, processing messages for one game session in a batch before moving to the next. This improves cache locality (all state for one game is hot in cache during its processing window) and simplifies per-game rate limiting. The poll group concept is internal to the relay server — clients don't know or care whether they share a relay with other games.
