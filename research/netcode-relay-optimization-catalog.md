# Netcode & Relay Optimization Catalog

> **Status:** Research document. Prioritized optimization recommendations for IC's relay server, network pipeline, and related infrastructure.
> **Date:** 2026-02

## 1. Executive Summary

This catalog identifies concrete optimizations for IC's relay server (ic-net), state hashing (ic-sim), and network pipeline. Recommendations are grouped by technique (cryptographic batching, incremental computation, multi-game server, wire format, client-side) with priority ratings and phase mappings.

## 2. Cryptographic Batch Operations

### 2.1 Ed25519 Batch Signature Verification

**Current:** Each relay-signed frame is verified individually with `ed25519_dalek::verify()`.

**Recommended:** Use `ed25519_dalek::verify_batch()` for relay frame verification. Batch verification uses a single multi-scalar multiplication instead of N individual verifications, achieving ~2× throughput for batches of 32+ signatures.

```rust
use ed25519_dalek::verify_batch;

fn verify_frame_batch(
    messages: &[&[u8]],
    signatures: &[Signature],
    public_keys: &[VerifyingKey],
) -> Result<(), SignatureError> {
    verify_batch(messages, signatures, public_keys)
}
```

**When to batch:** At the relay, accumulate incoming frames over a short window (e.g., 1ms or N frames, whichever comes first) and verify the batch together. On the client, batch-verify all frames received in a single network read.

**Priority:** High. Signature verification is per-frame overhead that scales linearly with player count and tick rate. Phase 3–4 (multiplayer implementation).

### 2.2 Serialize-Once, Encrypt-N Broadcast

**Current (implied):** The relay serializes and encrypts each outgoing frame per recipient.

**Recommended:** For broadcast frames (order bundles sent to all players in a game), serialize the frame payload once, then encrypt with each recipient's session key. The serialization (which involves TLV encoding of potentially complex order data) is the expensive part; AES-GCM encryption of the serialized bytes is comparatively cheap.

```rust
fn broadcast_frame(frame: &RelayFrame, recipients: &[SessionKey]) -> Vec<EncryptedFrame> {
    let serialized = frame.serialize(); // Once
    recipients.iter()
        .map(|key| key.encrypt(&serialized)) // N encryptions of same bytes
        .collect()
}
```

**Priority:** Medium. Most impactful for games with 4+ players where the same order bundle goes to everyone.

### 2.3 AES-NI Multi-Buffer (Future)

Intel's AES-NI multi-buffer library can encrypt multiple independent buffers in a single pass using AVX-512. This is a future optimization opportunity when IC supports 16+ player games where the broadcast fan-out is high enough to justify the complexity.

**Priority:** Low. Phase 5+ only if profiling shows encryption as a bottleneck.

## 3. Incremental Computation

### 3.1 Bevy Change Detection for State Hashing

**Current:** State hash is computed by hashing all sim-relevant ECS components every tick.

**Recommended:** Use Bevy's built-in change detection (`Changed<T>` query filter) to hash only components that actually changed since the last tick. Most entities don't change every tick — a unit standing still, a building producing, a tree existing — their hash contribution is stable.

```rust
fn incremental_state_hash(
    mut hasher: ResMut<IncrementalHasher>,
    changed_positions: Query<(Entity, &Position), Changed<Position>>,
    changed_health: Query<(Entity, &Health), Changed<Health>>,
    // ... other sim-relevant components
) {
    for (entity, pos) in changed_positions.iter() {
        hasher.update_component(entity, ComponentKind::Position, pos);
    }
    for (entity, hp) in changed_health.iter() {
        hasher.update_component(entity, ComponentKind::Health, hp);
    }
    // Final hash combines stable base hash + changed component hashes
}
```

The `IncrementalHasher` maintains a per-entity hash table. When a component changes, its entity's hash is recomputed and the global hash is updated. For a 500-unit game where ~20% of units move per tick, this reduces hashing work by ~3-4×.

**Priority:** High. State hashing runs every tick. Phase 2 (simulation implementation).

### 3.2 Relay Filter Chain Monomorphization

**Current (design):** The relay's composable filter chain (rate limiter → validator → behavioral analysis → broadcast) is trait-based with dynamic dispatch.

**Recommended:** For the standard lockstep filter chain (which is the same for all games), use a monomorphized `StandardFilterChain<V, S, R, L>` struct where the filter types are generic parameters, not trait objects. This eliminates vtable indirection in the hot path.

```rust
struct StandardFilterChain<V: Validator, S: SuspicionScorer, R: RateLimiter, L: LagDetector> {
    validator: V,
    scorer: S,
    limiter: R,
    lag_detector: L,
}

impl<V, S, R, L> FilterChain for StandardFilterChain<V, S, R, L>
where V: Validator, S: SuspicionScorer, R: RateLimiter, L: LagDetector
{
    fn process(&mut self, frame: &IncomingFrame) -> FilterResult {
        self.limiter.check(frame)?;
        self.validator.validate(frame)?;
        self.lag_detector.check(frame)?;
        self.scorer.score(frame)
    }
}
```

The trait object fallback remains available for custom/mod-provided filters.

**Priority:** Medium. Marginal per-frame improvement but compounds over high tick rates.

### 3.3 Ownership Pre-Check Bitset Cache

**Current:** Order validation checks unit ownership by querying the ECS (entity → Owner component → compare with order.player_id).

**Recommended:** Maintain a pre-computed bitset per player indicating which entities they own. Updated when entities are created, destroyed, or change ownership (rare events). The bitset makes ownership pre-check a single bit test instead of an ECS query.

```rust
struct OwnershipCache {
    /// owned_by[player_index] is a bitset of entity indices owned by that player
    owned_by: Vec<FixedBitSet>,
}

impl OwnershipCache {
    fn owns(&self, player: PlayerIndex, entity: EntityIndex) -> bool {
        self.owned_by[player.0 as usize].contains(entity.0 as usize)
    }
}
```

**Priority:** Medium. Ownership check is per-order overhead. Most orders target owned entities, so this is always the first check.

## 4. Multi-Game Server Optimizations

### 4.1 sendmmsg / io_uring Batched I/O

For `ic-server` hosting multiple simultaneous games, use OS-level batched I/O:

- **Linux:** `sendmmsg()` sends multiple UDP datagrams in a single syscall (reduces syscall overhead by ~10× for a server handling 20 games)
- **Linux 5.1+:** `io_uring` for fully asynchronous batched send/recv (further reduces context switches)
- **Windows:** Registered I/O (RIO) for similar gains

IC's networking layer should abstract this behind the existing Transport trait, with the batched implementation used automatically when available.

**Priority:** Medium. Phase 5 (dedicated server deployment).

### 4.2 Arena Allocation Per Game (bumpalo)

Each game session on a multi-game server has its own allocation pattern (frame buffers, order queues, scratch space). Using per-game arena allocators (`bumpalo`) avoids global allocator contention and enables bulk deallocation when a game ends.

```rust
use bumpalo::Bump;

struct GameSession {
    arena: Bump,
    // All per-tick scratch allocated from arena
    // Arena reset at end of each tick
}
```

**Priority:** Low-Medium. Only matters for servers hosting 10+ simultaneous games.

### 4.3 Cross-Game Suspicion Accumulation

**Current (design):** Behavioral analysis (Kaladin) operates per-game.

**Recommended:** Add a `ServerWideBehavioralState` that accumulates suspicion across all games on a server. A player who triggers low-level anomalies in multiple games is more suspicious than one anomaly in one game.

```rust
pub struct ServerWideBehavioralState {
    /// Per-player suspicion accumulated across all games on this server
    player_suspicion: HashMap<PlayerId, AccumulatedSuspicion>,
}

pub struct AccumulatedSuspicion {
    pub total_score: f64,
    pub game_count: u32,
    pub flagged_game_count: u32,
    pub last_game_tick: SimTick,
}
```

**Priority:** Medium. Phase 5+ (multi-game server deployment).

## 5. Wire Format Optimizations

### 5.1 Zero-Copy Frame Decoding

**Current (design):** Frame bytes are deserialized into owned Rust structs.

**Recommended:** Add a `FrameRef<'a>` type that borrows from the received byte buffer, parsing only the header (frame type, length, sequence number) and providing lazy accessors for the payload.

```rust
pub struct FrameRef<'a> {
    header: FrameHeader,     // Parsed (8 bytes, cheap)
    payload: &'a [u8],       // Borrowed from receive buffer
}

impl<'a> FrameRef<'a> {
    pub fn frame_type(&self) -> FrameType { self.header.frame_type }
    pub fn sequence(&self) -> u32 { self.header.sequence }
    
    /// Parse payload only when needed
    pub fn decode_orders(&self) -> Result<Vec<TimestampedOrder>> {
        TimestampedOrder::decode_batch(self.payload)
    }
}
```

The relay doesn't need to fully decode most frames — it forwards them. Only frames that need validation (rate limiting, behavioral analysis) are fully decoded. For pass-through frames, the relay reads the header and copies the raw payload bytes.

**Priority:** Medium-High. Relay throughput is directly limited by per-frame processing cost.

### 5.2 Pre-Computed TLV Templates

For common order types (Move, Attack, Build), pre-compute the TLV header bytes at startup. The per-order serialization then copies the template and fills in the variable fields (target coordinates, entity IDs).

**Priority:** Low. Minor per-order improvement.

## 6. Client-Side Optimizations

### 6.1 Pipelined Decryption + Deserialization

**Current (implied):** Client receives frame → decrypt → deserialize → apply.

**Recommended:** Pipeline the stages: while tick N's orders are being applied, decrypt tick N+1's frame and deserialize tick N+2's frame. On modern CPUs (even 2-core), AES-NI decryption is essentially free — the bottleneck is deserialization and application.

**Priority:** Low. Only matters if client-side frame processing becomes a bottleneck (unlikely for 2-8 player games).

### 6.2 Adaptive Sync Hash Depth

**Current:** Full state hash computed every tick for desync detection.

**Recommended:** In steady state (no desync detected for N ticks), reduce hash depth to RNG-state-only (cheapest possible check that catches the most common desync cause). On any hash mismatch, escalate to full state hash for detailed comparison.

```rust
enum HashDepth {
    RngOnly,       // Just the RNG state (1 hash, cheapest)
    EntityCount,   // RNG + entity counts per archetype
    Full,          // All sim-relevant state (expensive)
}
```

The escalation chain: RngOnly → EntityCount → Full. Each level narrows the desync location. Start at `Full` for the first 100 ticks (catch early desyncs), then settle into `RngOnly` if stable.

**Priority:** Medium. Reduces per-tick overhead in the common case (no desync).

## 7. Implementation Priority Summary

| Priority | Optimization | Section | Phase |
|---|---|---|---|
| **High** | Incremental state hashing (Bevy change detection) | §3.1 | Phase 2 |
| **High** | Ed25519 batch verification | §2.1 | Phase 3–4 |
| **High** | Zero-copy frame decoding | §5.1 | Phase 3 |
| **Medium-High** | Adaptive sync hash depth | §6.2 | Phase 2–3 |
| **Medium** | Serialize-once broadcast | §2.2 | Phase 3 |
| **Medium** | Filter chain monomorphization | §3.2 | Phase 3 |
| **Medium** | Ownership pre-check bitset | §3.3 | Phase 2 |
| **Medium** | sendmmsg / io_uring | §4.1 | Phase 5 |
| **Medium** | Cross-game suspicion | §4.3 | Phase 5+ |
| **Low-Medium** | Per-game arena allocation | §4.2 | Phase 5 |
| **Low** | AES-NI multi-buffer | §2.3 | Phase 5+ |
| **Low** | Pre-computed TLV templates | §5.2 | Phase 3 |
| **Low** | Pipelined decrypt/deserialize | §6.1 | Phase 5 |

## 8. Relationship to Design Documents

| Document | Section Affected | Nature of Change |
|---|---|---|
| `10-PERFORMANCE.md` | Section index | Add reference to optimization catalog for non-sim performance |
| `netcode/relay-architecture.md` | Filter chain, frame handling | Add monomorphized filter chain, zero-copy frames, batch crypto |
| `netcode/desync-recovery.md` | Sync hash | Add adaptive hash depth |
| `architecture/gameplay-systems.md` | Combat resolution | Add ownership bitset cache |
| `decisions/09b/D074-community-server-bundle.md` | Multi-game hosting | Add sendmmsg, arena allocation, cross-game suspicion |
