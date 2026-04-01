# Cross-Engine Compatibility Packs

> **Sub-page of:** [Cross-Engine Compatibility](../07-CROSS-ENGINE.md)
> **Status:** Design-ready. Phase 5 delivery: pack data model + replay import configuration (Level 1). Live cross-engine handshake and in-match translation (Level 2+) deferred pending Level 2+ scheduling.
> **Key decisions:** D011, D018, D019, D033, D043, D045

## Overview

A **CompatibilityPack** bundles all the configuration needed to make IC's simulation align with a specific foreign engine version. It connects IC's existing switchable subsystems — balance presets (D019), pathfinding presets (D045), and sim-affecting QoL toggles (D033) — into a coherent package that activates when replaying foreign replays (Phase 5, Level 1) or, in a future live cross-engine session (Level 2+, not yet scheduled), auto-activates during the protocol handshake.

Without compatibility packs, cross-engine play drifts into systematic desync because IC's default behavior (IC Default balance, IC pathfinding, IC production rules) diverges from the foreign engine's behavior. With compatibility packs, the switchable traits (D041) are set to their closest-matching configuration, reducing reconciler corrections to edge cases only.

## Data Model

```rust
pub struct CompatibilityPack {
    /// Which engine and version range this pack targets.
    pub target_engine: EngineTag,              // "openra", "cncnet", "remastered"
    pub target_version_range: VersionRange,    // e.g., ">=20231015"
    
    /// Match-level sim configuration to activate.
    /// Covers only the sim-affecting axes (balance preset, pathfinding preset,
    /// sim-affecting QoL toggles) — NOT per-player presentation settings
    /// (theme, render mode, client-only QoL) and NOT per-AI-slot commanders
    /// (D043 — see `recommended_ai_commander` below). This distinction
    /// follows the sim/client split in D019, D033, and D045: sim-affecting
    /// settings are whole-match (lobby-enforced), while presentation axes
    /// are per-player and AI selection is per-slot.
    pub match_config: MatchCompatibilityConfig,
    
    /// OrderCodec to use for wire translation (registered in OrderTypeRegistry).
    pub codec: OrderCodecId,
    
    /// Known behavioral baseline: expected sim behavior characteristics
    /// used by the reconciler to distinguish expected drift from bugs.
    pub behavioral_baseline: BehavioralBaseline,
    
    /// Known divergences that do NOT indicate bugs or cheating.
    /// The reconciler tolerates these without triggering desync alerts.
    pub known_divergences: Vec<KnownDivergence>,
    
    /// Expected correction rate (corrections per 100 ticks) under normal play.
    /// If actual rate exceeds this by 3×, flag the match for investigation.
    pub expected_correction_rate: f32,
    
    /// Validation status — has this pack been verified against replay corpus?
    pub validation_status: PackValidationStatus,
    
    /// Recommended AI commander for cross-engine matches using this pack.
    /// Advisory only — the host selects AI per slot in the lobby (D043).
    /// This recommendation helps the host pick an AI whose behavior
    /// approximates the foreign engine's built-in AI.
    pub recommended_ai_commander: Option<AiCommanderId>,
}

/// The sim-affecting subset of an experience profile.
/// An experience profile (D033) bundles six axes: balance, pathfinding, QoL,
/// AI, theme, and render mode. Only balance, pathfinding, and sim-affecting
/// QoL are whole-match lobby settings where all players must agree. Theme
/// and render mode are per-player presentation. AI presets are per-AI-slot
/// (D043) — the host selects an AI commander per slot, not a match-wide
/// AI setting — so they are NOT part of this struct.
pub struct MatchCompatibilityConfig {
    /// Balance preset to use (D019). Whole-match — all players must agree.
    pub balance_preset: BalancePresetId,
    /// Pathfinding preset to use (D045). Whole-match — all players must agree.
    pub pathfinding_preset: PathfindingPresetId,
    /// Sim-affecting QoL toggles (D033 production/commands/gameplay sections).
    /// Whole-match — all players must agree.
    pub sim_qol_overrides: Vec<(QolToggleId, bool)>,
}

pub struct BehavioralBaseline {
    /// Expected pathfinding characteristics (e.g., units may take wider paths)
    pub pathfinding_tolerance: PathfindingTolerance,
    /// Expected production timing variance (e.g., ±2 ticks on build completion)
    pub production_timing_tolerance_ticks: u32,
    /// Expected damage calculation variance (rounding differences)
    pub damage_rounding_tolerance: i32,
}

pub struct KnownDivergence {
    pub category: DivergenceCategory,
    pub description: String,
    pub severity: DivergenceSeverity,
}

pub enum DivergenceCategory {
    PathfindingTieBreaking,
    ProductionQueueOrdering,
    DamageRounding,
    RngSequenceDrift,
    HarvesterBehavior,
    ProjectileTracking,
}

pub enum PackValidationStatus {
    /// Verified against a corpus of 100+ replays from the target engine
    Verified { replay_count: u32, last_verified: String },
    /// Community-submitted, partially tested
    CommunityTested { tester_count: u32 },
    /// Untested — use at own risk
    Untested,
}
```

## Auto-Selection Pipeline

> **Phase delivery note:** The auto-selection pipeline below is the design for live cross-engine sessions (Level 2+, not yet scheduled). For Phase 5 (Level 1, replay import), pack selection is manual or inferred from the replay file's engine metadata.

When a foreign client connects to an IC-hosted game (or IC joins a foreign-hosted game), the compatibility pack is selected automatically during the protocol handshake:

```
1. ProtocolIdentification received
   → Extract engine tag + version from the handshake
   
2. Query CompatibilityPack registry
   → Find pack matching (engine_tag, version) with best validation_status
   
3. If found:
   → Activate match_config (sets balance, pathfinding, sim-affecting QoL)
   → Apply recommended_ai_commander as lobby default (host can override per-slot)
   → Register codec in OrderTypeRegistry
   → Configure reconciler with behavioral_baseline + known_divergences
   → Display pack info in lobby: "Using OpenRA v20231015 compatibility"
   
4. If NOT found:
   → Display warning: "No compatibility pack available for [engine] [version]"
   → Offer to proceed with IC defaults (expect high desync rate)
   → Or suggest downloading a pack from Workshop
```

## Handshake Integration

> **Phase delivery note:** The handshake extensions below are the design for live cross-engine sessions (Level 2+). For Phase 5 (Level 1), replay import uses `ForeignReplayPlayback` which does not require a live handshake.
>
> **Schema ownership:** The canonical `VersionInfo` is defined in `security/vulns-mods-replays.md` § Vulnerability 10 and includes `engine_version`, `sim_hash`, `mod_manifest_hash`, and `protocol_version`. Those fields remain unchanged and are used for native IC-to-IC version matching. The struct below is a **proposed foreign-handshake envelope** — a separate type that wraps the existing handshake with cross-engine fields. It does NOT replace `VersionInfo`. If accepted for Level 2+, it would live alongside `VersionInfo` in the handshake sequence; this page documents the proposed shape. Do not treat this struct as authoritative.

The cross-engine handshake (Level 2+) would use a **separate envelope** alongside the canonical `VersionInfo`:

```rust
/// Foreign-handshake envelope — sent IN ADDITION TO the canonical
/// VersionInfo (which carries sim_hash, mod_manifest_hash, etc.).
/// Native IC clients send only VersionInfo. Foreign clients send
/// both VersionInfo (best-effort) and ForeignHandshakeInfo.
pub struct ForeignHandshakeInfo {
    pub engine: EngineTag,                   // "openra", "cncnet", "remastered"
    pub engine_version: String,              // foreign engine's version string
    pub game_module: GameModuleId,           // which game (RA1, TD, TS, etc.)
    pub mod_profile_fingerprint: Option<[u8; 32]>,  // D062 profile hash (if available)
    pub capabilities: Vec<Capability>,       // what the foreign client supports
}
```

The `game_module` field is critical — an OpenRA RA1 game has different compatibility requirements than an OpenRA TD game, even at the same engine version.

### Foreign-Client Exception to the Version Gate

The canonical version gate (`vulns-mods-replays.md` § Vulnerability 10) requires **exact equality** on `sim_hash` and `mod_manifest_hash` and uses `VersionInfo` to filter incompatible games from the server browser. This rule is correct for native IC-to-IC connections and remains unchanged.

Foreign clients cannot satisfy this gate — their engine has no IC sim binary to hash, and their mod manifest format differs. The cross-engine handshake handles this explicitly:

1. **Foreign clients send `VersionInfo` with zeroed `sim_hash` and `mod_manifest_hash`.** The relay detects the zeroed fields alongside a present `ForeignHandshakeInfo` and routes the connection through the **cross-engine admission path** instead of the native version gate.
2. **Cross-engine admission** skips `sim_hash`/`mod_manifest_hash` equality checks and instead requires: (a) a matched `CompatibilityPack` for the foreign engine/version, (b) `protocol_version` compatibility, and (c) host has set `max_foreign_tier ≥ 1` (relay-security.md § Lobby Trust UX).
3. **Server browser filtering:** Games hosted with `max_foreign_tier = 0` (default for ranked) are invisible to foreign clients. Games with `max_foreign_tier ≥ 1` appear in cross-engine listings with an engine badge and trust tier indicator.
4. **Native clients are never affected.** The zeroed-field exception triggers only when `ForeignHandshakeInfo` is also present. A native IC client sending zeroed hashes without `ForeignHandshakeInfo` is rejected by the standard gate (patched/corrupted binary).

This exception path preserves the canonical version-mismatch defense for native play while enabling controlled cross-engine admission.

## Order Vocabulary Intersection

Before a cross-engine match begins, IC computes a `CompatibilityReport` showing which orders can be translated:

```rust
pub struct CompatibilityReport {
    /// Orders that translate perfectly between engines
    pub fully_compatible: Vec<OrderTypeId>,
    /// Orders that translate with approximation (e.g., waypoint count differs)
    pub approximate: Vec<(OrderTypeId, String)>,  // (order, approximation note)
    /// Orders that cannot be translated (game-specific features)
    pub incompatible: Vec<(OrderTypeId, String)>,  // (order, reason)
    /// Overall compatibility percentage
    pub compatibility_score: f32,
}
```

This report is displayed in the pre-join confirmation screen (see [Relay Security Architecture](relay-security.md)). Players see exactly what works and what doesn't before committing to a cross-engine match.

### OrderTypeRegistry

> **Relationship to existing design:** The existing `OrderCodec` trait in `07-CROSS-ENGINE.md` handles single-codec translation. `OrderTypeRegistry` is a proposed evolution for multi-codec dispatch when multiple game modules or mods register independent codecs. Not required for Phase 5 Level 1 (replay import uses a single codec per `ForeignReplayPlayback` session).

Mod-aware translator registration:

```rust
pub struct OrderTypeRegistry {
    /// Codecs registered by game module
    module_codecs: HashMap<GameModuleId, Vec<Box<dyn OrderCodec>>>,
    /// Codecs registered by compatibility packs
    pack_codecs: HashMap<CompatibilityPackId, Vec<Box<dyn OrderCodec>>>,
    /// Codecs registered by mods (for mod-specific orders)
    mod_codecs: HashMap<ModId, Vec<Box<dyn OrderCodec>>>,
}
```

When translating an order, the registry checks: pack codec → module codec → mod codec → fail with `TranslationResult::Untranslatable`.

## Translation Fidelity

Every translated order carries a fidelity assessment:

```rust
pub enum TranslationResult {
    /// Perfect 1:1 translation, no information loss
    Exact(TranslatedOrder),
    /// Translated with approximation — order is valid but not perfectly equivalent
    Approximated {
        order: TranslatedOrder,
        loss: SemanticLoss,
        description: String,
    },
    /// Cannot translate — order type doesn't exist in target engine
    Untranslatable {
        original: OrderTypeId,
        reason: String,
    },
}

pub enum SemanticLoss {
    /// Minor: target receives slightly different parameters (e.g., waypoint snapped to grid)
    Minor,
    /// Moderate: order intent preserved but execution will differ noticeably
    Moderate,
    /// Major: order intent partially lost (fallback to nearest equivalent)
    Major,
}
```

### Live Translation Health Indicator

During a cross-engine match, the UI displays a `TranslationHealth` indicator:

```rust
pub struct TranslationHealth {
    /// Rolling average translation fidelity (0.0 = all failures, 1.0 = all exact)
    pub fidelity_score: f32,
    /// Number of approximated orders in last 100 ticks
    pub recent_approximations: u32,
    /// Number of untranslatable orders in last 100 ticks  
    pub recent_failures: u32,
    /// Trend: improving, stable, or degrading
    pub trend: HealthTrend,
}
```

Displayed as a small icon in the match UI (green/yellow/red). Players can expand it to see detailed translation statistics.

## Mid-Game Translation Failure Protocol

When an order cannot be translated during gameplay:

```rust
pub enum TranslationFailureResponse {
    /// Drop silently — ONLY for non-gameplay events that have no sim effect
    /// (e.g., camera movement, chat, UI-only pings). Gameplay orders must
    /// never use SilentDrop — see gotcha #8 in 07-CROSS-ENGINE.md.
    SilentDrop,
    /// Substitute a safe fallback (e.g., untranslatable ability → Stop command)
    Substitute { fallback: TranslatedOrder, reason: String },
    /// Notify the player and skip — significant orders
    NotifyAndSkip { reason: String },
    /// Escalate — critical failure (e.g., synchronization-relevant order type unknown)
    Escalate { trigger_reconciler: bool },
}
```

The response strategy is defined per order category in the compatibility pack. Critical orders (build, attack, move) always `Escalate`. Non-gameplay events (camera movement, chat) use `SilentDrop`. All other gameplay orders must use `Substitute`, `NotifyAndSkip`, or `Escalate` — never `SilentDrop`.

## Workshop Distribution

> **Proposed category.** `CompatibilityPack` is not yet listed in the canonical Workshop resource categories (D030). Adding it requires a D030 category update when Level 2+ cross-engine play is formally scheduled. Until then, packs can be distributed as tagged `Mod` resources with a `cross-engine-compat` tag.

Compatibility packs would be a Workshop resource category:

- **Proposed category:** `CompatibilityPack` (pending D030 update)
- **Metadata:** target engine, version range, validation status, last-verified date
- **Dependencies:** May depend on mod data packs (e.g., a Combined Arms compatibility pack depends on the Combined Arms mod data)
- **Versioning:** Semver, tied to both IC version and target engine version
- **Validation:** Community testing infrastructure — packs include a replay corpus and automated verification scores

## Pre-Join Risk Assessment

Before entering a cross-engine lobby, IC displays a confirmation screen:

```
┌─────────────────────────────────────────────────────────────────┐
│  Cross-Engine Match: OpenRA v20231015                           │
│                                                                  │
│  Compatibility Pack: openra-ra1-2023 (Verified, 847 replays)   │
│  Compatibility Score: 94% (23 of 245 order types approximate)  │
│                                                                  │
│  ⚠ Known Limitations:                                           │
│  • Pathfinding tie-breaking may differ (cosmetic)               │
│  • Chrono Shift targeting uses grid snap (minor)                │
│  • Custom mod orders not supported                              │
│                                                                  │
│  ⚠ Security:                                                    │
│  • Fog-of-war protection: Full (translator-authoritative)       │
│  • Maphack protection: Active decoys                            │
│  • The foreign client cannot be verified as unmodified           │
│                                                                  │
│  [Join Match]  [View Full Report]  [Cancel]                     │
└─────────────────────────────────────────────────────────────────┘
```

## Cross-Engine Replay Provenance

> **Schema ownership:** The canonical `.icrep` metadata schema is defined in `formats/save-replay-formats.md` § Metadata (JSON). Foreign replay CLI conversion uses a `converted_from` block defined in `decisions/09f/D056-replay-import.md`. The struct below is a **proposed extension** for live cross-engine matches (Level 2+, not yet scheduled). If accepted, these fields would be added to the canonical schema in `save-replay-formats.md` as an optional `cross_engine` block — this page documents the proposed shape. Do not treat this struct as authoritative. For Phase 5 Level 1 (replay import), the existing `converted_from` provenance in D056 is sufficient.

Replays from live cross-engine matches would include additional metadata:

```rust
pub struct CrossEngineReplayMeta {
    /// Engines involved in this match
    pub engines: Vec<EngineParticipant>,
    /// Compatibility pack used
    pub compatibility_pack: CompatibilityPackId,
    /// Translation fidelity log (per-tick summary, not per-order)
    pub translation_summary: TranslationSummary,
    /// Translation failure log
    pub translation_failures: Vec<TranslationFailureRecord>,
    /// Authority mode used (if fog-authoritative)
    pub authority_mode: Option<CrossEngineAuthorityMode>,
    /// Reconciliation corrections applied (Level 2+)
    pub corrections_applied: u32,
}
```

This metadata is included in the replay file alongside `CertifiedMatchResult` and visible in the replay viewer's info panel.

## Implementation Phase

| Component | Phase | Priority | Notes |
|---|---|---|---|
| `CompatibilityPack` data model | Phase 5 | Critical | Used by Level 1 replay import |
| `game_module` in handshake | Phase 2 | High | Native IC handshake field |
| Pack-based replay import config | Phase 5 | Critical | Level 1: `ForeignReplayPlayback` loads pack for sim alignment |
| Auto-selection pipeline | Level 2+ (not yet scheduled) | Critical | Live cross-engine sessions only |
| `CompatibilityReport` + pre-join UI | Level 2+ (not yet scheduled) | High | Live cross-engine sessions only |
| `TranslationHealth` live indicator | Level 2+ (not yet scheduled) | Medium | Live cross-engine sessions only |
| Workshop distribution | Phase 5+ | Medium | Packs useful for replay import even without live play |
| Replay provenance metadata | Phase 5+ | Medium | Extends existing `converted_from` (D056) |
| Mid-game failure protocol | Level 2+ (not yet scheduled) | Medium | Live cross-engine sessions only |

## Architectural Compliance

- **Sim unchanged:** The compatibility pack selects *which* already-registered `Pathfinder`, balance preset, and QoL configuration the sim uses, via existing trait-based switchability (D041). No new sim-level code.
- **Relay unchanged:** The relay remains a lightweight order router. Translation happens in the `ProtocolAdapter` layer (ic-net), not in the relay core.
- **NetworkModel boundary preserved:** The compatibility pack is configuration data, not a new NetworkModel.
