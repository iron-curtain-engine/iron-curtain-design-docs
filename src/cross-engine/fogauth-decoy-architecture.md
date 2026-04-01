# FogAuth & Decoy Architecture for Cross-Engine Play

> **Sub-page of:** [Cross-Engine Compatibility](../07-CROSS-ENGINE.md)
> **Status:** Design-ready architecture. Native FogAuth implementation: M11 (`P-Optional`, pending P007 — client game loop strategy). Cross-engine translator-authoritative model: depends on native FogAuth.
> **Depends on:** `research/fog-authoritative-server-design.md` (native FogAuth design), P007 resolution (game loop variant for FogAuth clients)

## Problem Statement

In lockstep multiplayer, every client has full game state in memory. A modified client can render enemy positions through fog-of-war (maphack). For native IC clients, the FogAuth server model prevents this by withholding fogged data entirely. But for cross-engine play, IC cannot control the foreign client's binary — the foreign client's lockstep partner is IC's translator, and the translator must send *something* for fogged areas or the foreign client desyncs.

The translator-authoritative model solves this: instead of sending nothing (which breaks lockstep) or sending real data (which enables maphack), the translator sends **fabricated decoy data** for fogged entities.

## Core Architecture

### Translator as Sole Interface

When IC hosts a cross-engine match, the `AuthoritativeRelay` runs:

```rust
pub struct AuthoritativeRelay {
    /// The real, authoritative simulation
    sim: SimState,
    /// Per-player visibility computation
    visibility: VisibilityComputer,
    /// Decoy generator for cross-engine fog protection
    decoys: DecoyGenerator,
    /// Per-foreign-client translator sessions
    translators: HashMap<PlayerId, TranslatorSession>,
    /// Standard relay core for order routing
    relay_core: RelayCore,
}
```

The foreign client's lockstep partner is NOT another player's sim — it's IC's translator. The translator runs the real sim, computes per-player visibility, and produces a **curated order stream** per foreign client.

### Two Parallel States

For each foreign client, IC maintains:
1. **Real state:** The authoritative sim (same as all IC clients see)
2. **Curated stream:** The order stream sent to the foreign client, containing:
   - Real orders for entities the foreign player can see (in their fog-of-war vision)
   - Decoy orders for entities the foreign player cannot see

### Curated Order Types

```rust
pub enum CuratedOrder {
    /// Real order from a visible entity — forwarded unmodified
    Real(TranslatedOrder),
    /// Ghost order for a fogged entity — fabricated by DecoyGenerator
    Decoy(TranslatedOrder),
    /// Order withheld — entity is fogged, no decoy needed (e.g., entity is idle)
    Withheld,
}
```

## Decoy Generation Tiers

### Tier 0 — Stale Orders (baseline)

When an entity exits vision, the translator **stops sending real orders** and either:
- Sends no orders (entity appears frozen at last-known position on the foreign client), or
- Repeats the last known order (entity appears to continue its last action)

Cheapest to implement. Provides basic fog protection but a sophisticated observer can notice entities "freezing" when they leave vision.

### Tier 1 — Last-Known Freeze

The foreign client sees entities at their last-known position. No new orders are generated. This is equivalent to what a maphacking player would see in a native game with FogAuth — entities simply don't exist outside vision.

The difference: in native FogAuth, fogged entities are removed from the client's ECS. In cross-engine play, fogged entities persist on the foreign client with stale state.

### Tier 2 — Plausible Ghost Behavior

The `DecoyGenerator` produces **plausible-looking fake orders** that match the entity type's expected behavior:

```rust
pub enum GhostBehavior {
    /// Unit patrols between last-known position and nearby waypoints
    Patrol { waypoints: Vec<WorldPos>, speed: FixedPoint },
    /// Harvester drives to nearest (last-known) resource field and back
    Harvest { resource_pos: WorldPos, refinery_pos: WorldPos },
    /// Building appears to be producing (animation, queued unit ghosts)
    StaticBuilding { production_visible: bool },
    /// Fake base expansion — MCV deploys, construction starts
    FakeExpand { target_pos: WorldPos },
}
```

Ghost behavior is entity-type-appropriate:
- **Infantry/vehicles:** Patrol between plausible positions
- **Harvesters:** Follow harvest cycles to known resource fields
- **Buildings:** Static, may appear to be producing
- **MCVs (rare):** May appear to be relocating or expanding

The ghost orders must be **valid in the foreign engine's protocol** — the compatibility pack's behavioral baseline informs which orders the foreign engine accepts.

### Tier 3 — Adversarial Counterintelligence (Phase 6+)

Active disinformation:
- Fake army movements to draw enemy scouts or premature attacks
- Decoy expansions at secondary bases to split enemy attention
- Fabricated production queues suggesting a tech switch

Tier 3 is opt-in and only available in casual/friendly matches where both players agree. It's explicitly a "fun mode" — not for any competitive context.

## Fog-Lift Transitions

When a fogged entity enters a foreign player's vision, the ghost must transition to the real entity seamlessly:

```
Tick N:   Entity outside vision → ghost at position G, patrolling
Tick N+1: Entity enters vision → real position is R (may differ from G)
Tick N+2: Translator sends Move order from G → R (2–5 tick transition)
Tick N+3: Entity reaches R, now receiving real orders
```

The transition is indistinguishable from a unit being given new commands. No desync, no teleportation, no protocol violation. OpenRA's sync hashing, original RA's simple lockstep, and CnCNet's tunnel relay all handle Move orders correctly.

**Pre-correction window:** The translator starts sending the real position 2–3 ticks before the entity enters vision, so the ghost "naturally" moves toward the real position before the player sees it.

## Security Analysis

### Directional Trust Asymmetry

The data flows are fundamentally asymmetric:

| Direction | What Flows | Who Controls |
|---|---|---|
| Foreign → IC | **Orders only** | Foreign client generates, IC validates |
| IC → Foreign | **State** (curated) | IC translator generates, foreign client renders |

The foreign client can only control its own units — every order is validated by IC's real sim. The translator controls the foreign client's entire perception of non-owned entities. A malicious foreign client cannot:
- Inject orders for units it doesn't own (IC validates ownership)
- Speed-hack (relay owns the clock)
- Lag-switch (missed deadline = idle order injected)
- See through fog (fog shows ghosts, not real positions)

A malicious foreign client CAN:
- Send adversarial order patterns (mitigated by `ForeignOrderFirewall`)
- Refuse to play (disconnects, handled by standard flow)

### What a Sophisticated Attacker Could Do

**Statistical ghost detection:** Recording ghost positions across many games could reveal patterns (e.g., ghosts always patrol in circles). Mitigated by varying ghost parameters per match using the commit-reveal game seed as RNG input.

**Timing analysis:** Comparing timing of visible vs. fogged orders in the curated stream. Mitigated by inserting ghost orders with timing characteristics matching real orders from the same player.

### Trust Model

The fundamental assertion: **IC's authority server is honest.** If compromised, the decoy system could feed disinformation to unfairly advantage IC players. Mitigated by existing relay trust infrastructure (signing keys, community server reputation).

## Configuration — Per-Match Toggle

```rust
pub enum CrossEngineFogProtection {
    /// Full translator-authoritative fog with decoy generation. Default.
    Full,
    /// FogAuth without decoys. Foreign clients receive no *new real data* for
    /// fogged entities — entities freeze at their last-visible position until
    /// they re-enter the foreign client's fog boundary. This is a state freeze,
    /// not literal silence (which would break lockstep). May cause visible
    /// "pop-in" on fog lift as frozen entities snap to current positions.
    FogAuthOnly,
    /// No fog protection. Full lockstep — both sides see everything.
    /// Foreign client CAN maphack. Requires explicit player consent.
    Disabled,
}
```

**Cross-engine ranked policy:** Cross-engine live play (Level 2+) is unranked by default — any promotion to ranked status requires a separate certification decision covering trust model, authority attestation, and enforcement (see `07-CROSS-ENGINE.md` § Anti-cheat warning). Within that unranked context, `Full` is the default and recommended setting. The toggle applies to casual/unranked cross-engine matches only.

**Player consent:** Reducing fog protection below `Full` requires:
1. Host confirmation (with typed "I understand" for `Disabled`)
2. All IC players in the lobby must confirm ("I Accept the Risk — Stay" / "Leave Lobby")

## Foreign Order Firewall

The `ForeignOrderFirewall` adds cross-engine-specific anomaly **scoring** between structural validation and broadcast. It is strictly advisory — it feeds suspicion scores into the existing Kaladin behavioral analysis pipeline but **never blocks or drops orders**. The canonical enforcement model is: relay performs structural validation only; behavioral analysis produces detection/evidence, not heuristic suppression; deterministic order rejection happens in the sim (D012). See `relay-security.md` § Trust Tier table, `vulns-client-cheating.md` § Behavioral Detection, and `vulns-infrastructure.md` § Defense.

```rust
pub struct ForeignOrderFirewall {
    pattern_buffer: RingBuffer<OrderPattern>,
    detectors: Vec<Box<dyn AnomalyDetector>>,
}

pub enum AnomalyResult {
    Clean,
    Suspicious { reason: String, score: f64 },
}
```

**Detectors** (scoring-only — feed into Kaladin `behavioral_score`):
- **Build-cancel oscillation:** Flags repeated queue/cancel of same building type (economic exploitation)
- **Pathfinding probe:** Flags systematic Move orders covering map in grid pattern (automated map scanning)
- **Rapid retarget:** Flags Attack orders changing target faster than human reaction time (bot detection)
- **Correction bias:** Flags reconciliation corrections that *only* benefit the foreign player (tampering indicator)

All detectors produce a score contribution. The relay **always forwards the order** to IC clients for deterministic sim validation (D012). The firewall's scores compound with standard Kaladin behavioral scores for post-hoc analysis and, if thresholds are breached, the ranking authority (not the relay) issues anti-cheat actions.

## Deployment Modes

| Mode | FogAuth | Decoys | Use Case |
|---|---|---|---|
| Casual lockstep (no FogAuth) | No | No | Trusted LAN / friends |
| Ranked (native IC) | Yes | N/A | Competitive IC-vs-IC |
| Cross-engine IC-hosted | Yes | Yes (Tier 0–2) | IC hosts, foreign joins |
| Cross-engine foreign-hosted | No | No | IC joins foreign game |

When IC joins a foreign-hosted game, fog protection depends on the foreign engine's capabilities. IC cannot protect itself from a foreign host's maphack — the host controls the sim. This is disclosed in the pre-join risk assessment.

## Implementation Phases

Native FogAuth is canonically deferred to M11 (`P-Optional`, pending P007 — client game loop strategy). The translator-authoritative cross-engine model depends on native FogAuth. This page documents the architecture so it is design-ready when M11 work begins.

| Milestone | What Ships | Scope |
|---|---|---|
| M11 | Relay-headless `ic-sim` for native FogAuth | Ranked/competitive maphack protection (IC-vs-IC). Requires `FogAuthGameLoop` or equivalent client loop variant (P007). |
| M11+ | Translator-authoritative model | Cross-engine: Tier 0 fog protection (depends on native FogAuth) |
| M11+ | Ghost behavior per entity type | Cross-engine: Tier 1 + Tier 2 decoys |
| M11+ | Adversarial decoys (if community demand) | Cross-engine: Tier 3 counterintelligence |

> **Note on earlier delivery:** The architecture would permit bringing FogAuth forward if P007 resolves earlier, but the canonical schedule is M11. The server-side capability infrastructure (`ic-server` binary, capability flags) supports FogAuth from day one — only the client-side game loop variant is the blocking design question. See `architecture/game-loop.md`, `netcode/network-model-trait.md`, and `research/fog-authoritative-server-design.md`.

## Architectural Compliance

- **ic-sim unchanged:** The sim is not aware of fog protection, decoys, or cross-engine play. It runs the same deterministic tick regardless.
- **NetworkModel boundary preserved:** FogAuth is a deployment mode of `ic-server`, not a sim modification. The sim produces world state; the server/translator decides what each client sees.
- **Relay architecture extended:** `AuthoritativeRelay` is a new relay mode (alongside standard lockstep relay) that adds headless sim + per-client visibility + per-client order curation.
