# Community Ecosystem & Editor Study — CnCNet, OpenRA, Remastered

> **Status:** Research document. Informs IC design across D038 (Scenario Editor), D040 (Asset Studio), D066 (Cross-Engine Export), and the cross-engine compatibility strategy.
> **Date:** 2026-02

## 1. Executive Summary

Three parallel C&C community ecosystems operate today. Each makes different trade-offs between authenticity, modernization, and accessibility. IC's opportunity is to bridge the gaps between all three without fragmenting any of them.

## 2. Ecosystem Profiles

### 2.1 CnCNet — Preservation & Competitive Play

- **Philosophy:** Keep the original games alive with minimal modification
- **Player base:** ~5,500 peak concurrent (across RA1, TD, TS, RA2, YR); primarily competitive/ladder players
- **Architecture:** 5-layer stack — IRC lobby → tunnel servers → spawner (injects into original binaries) → ladder API → map API
- **Strengths:** Authentic gameplay feel; established competitive scene with active tournaments; CnCNet tunnel servers solve NAT traversal for original binaries
- **Weaknesses:** Tied to original binaries (Windows-only for most games); limited modding beyond map rotation; IRC-based lobby is dated
- **Key insight for IC:** CnCNet's tunnel architecture is a reference for cross-engine connectivity. The spawner approach (injecting into running binaries) demonstrates the value of "be indistinguishable from the original" — players care about authenticity

### 2.2 OpenRA — Modernization & Modding

- **Philosophy:** Reimplementation with modern QoL and expanded modding
- **Player base:** Smaller concurrent than CnCNet but larger total installs; skews toward modders and casual players
- **Architecture:** C# engine, trait-based entity system, integrated map editor, MiniYAML data pipeline
- **Strengths:** Cross-platform; active modding scene (Combined Arms, Romanov's Vengeance); integrated editor with play-test button; random map generation in lobby
- **Weaknesses:** Campaign support incomplete after 18+ years of development; gameplay diverges from original in ways that alienate purists; performance limits at scale (~800 units before slowdown); C# GC pauses
- **Key insight for IC:** OpenRA's map editor demonstrates that in-engine editing with instant play-testing dramatically lowers the barrier to content creation. The random map generator (even without LLM) is a high-value feature for replayability

### 2.3 Remastered Collection — Official HD Restoration

- **Philosophy:** EA's official HD update with original gameplay preserved
- **Player base:** Steam player counts; primarily single-player campaign and casual multiplayer
- **Architecture:** Original C++ engine DLLs (GPL v3 released) + C# wrapper for HD rendering + Steam integration
- **Strengths:** Official product with EA backing; HD sprites, remastered audio; campaign fully playable; GPL source release enables community engine work
- **Weaknesses:** Windows-only; no cross-platform; limited modding (no editor, no scripting); multiplayer population declining
- **Key insight for IC:** The GPL source release of the C++ engine DLLs is IC's most valuable reference material. The Remastered Collection proves there's market demand for "the original game, but better" — IC should target this audience directly

## 3. Editor Feature Comparison

### 3.1 Matrix: EA FinalSun/FA2 vs WAE vs OpenRA Editor vs IC SDK

| Feature | FinalSun/FA2 (EA GPL) | WAE (CnCNet) | OpenRA Editor | IC SDK (Planned) |
|---|---|---|---|---|
| **Platform** | Windows (MFC C++) | Windows (DirectX 11) | Cross-platform | Cross-platform (Bevy) |
| **Trigger editing** | Tab-based (events/actions separate screens) | Unified view (events+actions on one screen) | Lua script files | Visual + Lua hybrid (D038) |
| **Waypoint management** | Manual number assignment | Two-way cross-referencing | Named markers | Named + typed (D038) |
| **Validation** | MapValidator::CheckMap() (compile-time) | Semantic validation (real-time) | Basic validation | Multi-layer (syntax → semantic → balance) |
| **Terrain generation** | None | Procedural generation tools | Random map gen in lobby | Procedural + LLM (Phase 7) |
| **Play testing** | Separate launch | Separate launch | In-editor play button | In-editor play (Bevy hot-reload) |
| **Data format** | Binary .map + .ini | Binary .map + .ini | .oramap (YAML + zip) | .icmap (YAML + assets, D067) |
| **Game switching** | Compile-time #ifdef | Runtime selection | Per-mod | Runtime GameModule trait (D018) |
| **Rendering** | Software renderer (166K lines IsoView.cpp) | DirectX 11 hardware | OpenRA's OpenGL | wgpu via Bevy |
| **Undo system** | 64-step undo | Unlimited undo | Basic undo | Command-pattern undo (D038) |

### 3.2 WAE Lessons (World Altering Editor — CnCNet)

WAE represents the most advanced community editor for the TS/RA2 family. Key innovations:

1. **Unified trigger view:** Events and actions displayed together on one screen, solving FinalSun's tab-switching pain. IC's trigger editor (D038) should adopt this pattern from day one
2. **Two-way waypoint cross-referencing:** Click a waypoint → see all triggers that reference it. Click a trigger → see all waypoints it uses. IC should extend this to all entity cross-references
3. **Semantic validation:** Real-time validation that catches logical errors (orphaned triggers, impossible conditions, unreachable map areas), not just syntax errors. Maps to IC's multi-layer validation (D038)
4. **Smart defaults:** New triggers pre-populate with sensible defaults based on context (e.g., creating a trigger near a building pre-selects building-related events). IC should learn from this UX pattern
5. **Terrain generation tools:** Procedural terrain generation within the editor. IC's Phase 5 non-LLM random map generator should provide similar inline generation

### 3.3 OpenRA Editor Lessons

1. **In-engine play button:** The single most impactful editor feature. Pressing play immediately launches the map in the game engine. Zero export/compile step. IC's Bevy-based editor inherits this naturally (editor and game share the renderer)
2. **Random map generation in lobby:** Available at game creation time, not just in the editor. Ensures even non-modders benefit from procedural content
3. **One-click Workshop upload:** Low-friction content sharing. IC's Workshop integration (D030) should match or exceed this

### 3.4 EA GPL Source Lessons (FinalSun/FA2)

Direct code study of the GPL-released editor source code reveals engineering patterns:

1. **MapValidator::CheckMap():** Structured validation in the original EA code. IC's validator should cover the same cases
2. **FAData.ini:** Data-driven trigger/action definitions — the editor reads its own trigger vocabulary from an INI file, making it extensible. IC's YAML-driven trigger system (D038) follows this same principle
3. **Bitmap-to-map converter:** EA included a bitmap heightmap import tool. IC should support heightmap import (PNG → terrain)
4. **Compile-time game switching:** FinalSun used `#ifdef` to switch between TS and RA2 behavior at compilation. IC's runtime `GameModule` trait (D018) is the correct evolution of this pattern
5. **Software renderer (166K lines):** IsoView.cpp is 166,000 lines of software rendering. This is the single largest file in the EA source. IC's Bevy/wgpu rendering avoids this entirely
6. **Tunnel tube editing rewrite (v2.0):** EA rewrote the tunnel tube editor from scratch in version 2.0, indicating it was one of the hardest editing interactions to get right. IC should plan for tunnel/underground editing complexity

## 4. CnCNet as Cross-Engine Target

### 4.1 Integration Architecture

CnCNet's 5-layer architecture provides a natural integration surface:

- **IRC Lobby layer:** `CnCNetIrcBridge` implements `CommunityBridge` trait. IC appears as games in CnCNet's lobby. This is NOT a tracking server — CnCNet is a community bridge
- **Tunnel layer:** IC does NOT use CnCNet tunnels (IC has its own relay). The bridge is lobby-level only
- **Spawner layer:** Not applicable (IC is a native engine, not injecting into original binaries)
- **Ladder API:** IC players can participate in CnCNet ladders if cross-engine play reaches Level 2+ (requires `CnCNetOrderCodec` that speaks original Westwood protocol)
- **Map API:** IC can fetch maps from CnCNet's shared map repository via the `CommunityBridge::fetch_map()` interface

### 4.2 Compatibility Requirements

For CnCNet cross-engine play to work, the `CompatibilityPack` concept (from cross-engine gap analysis) must include:
- **`classic-ea` balance preset** matching the original Westwood balance exactly (not OpenRA's rebalanced values)
- **Westwood-protocol `OrderCodec`** that encodes IC orders into original Westwood wire format
- **CnCNet trust model:** CnCNet servers are `VerifiedForeign` (Tier 1 in relay-security.md trust hierarchy)

### 4.3 Deep Links

IC should register `ic://` URI scheme with sub-schemes:
- `ic://cncnet/<channel>` — join a CnCNet game channel
- `ic://openra/<lobby-id>` — join an OpenRA lobby
- `ic://direct/<host:port>` — direct connect

## 5. IC Positioning: Filling the Gaps

| Gap | CnCNet | OpenRA | Remastered | IC |
|---|---|---|---|---|
| Cross-platform | ❌ (mostly) | ✅ | ❌ | ✅ Planned |
| Campaign support | ✅ (original) | ⚠ (incomplete) | ✅ (original) | ✅ Planned (branching, D021) |
| Modern editor | ❌ | ⚠ (basic) | ❌ | ✅ Planned (D038 SDK) |
| Competitive infrastructure | ✅ (CnCNet ladder) | ⚠ (basic) | ⚠ (Steam) | ✅ Planned (D055 ranked) |
| Modding depth | ❌ (limited) | ✅ (YAML + C#) | ❌ | ✅ Planned (YAML → Lua → WASM) |
| Performance at scale | ✅ (original C++) | ⚠ (C# GC) | ✅ (original C++) | ✅ Planned (Rust, D015) |
| Community bridges | N/A (is the community) | Own browser | Steam only | All three (CommunityBridge) |

## 6. Non-LLM Random Map Generation

Before Phase 7's LLM-powered generation, IC needs a procedural random map generator (Phase 5) for lobby use:

```rust
pub struct MapGeneratorParams {
    pub size: MapSize,
    pub players: u8,
    pub terrain_profile: TerrainProfile,  // e.g., "island", "desert", "arctic"
    pub symmetry: MapSymmetry,            // Rotational2/4, Mirror, None
    pub resource_density: ResourceDensity,
    pub water_fraction: FixedPoint,
    pub cliff_density: CliffDensity,
    pub seed: u64,
}
```

Key requirements:
- Deterministic (same seed → same map, for lobby sync)
- Symmetry enforcement (competitive maps must be fair)
- Resource placement respects symmetry (equal ore/gem patches per starting position)
- Terrain respects pathability (no isolated start positions)
- Outputs standard IC map format (not a special "generated" format)

This bridges the gap between CnCNet (fixed map pool) and OpenRA (has random gen) for IC's lobbies.

## 7. Relationship to Design Documents

| Document | Relevance |
|---|---|
| `decisions/09f/D038-scenario-editor.md` | Editor architecture directly informed by WAE and FinalSun analysis |
| `decisions/09f/D040-asset-studio.md` | Asset browser patterns from WAE |
| `decisions/09c/D066-cross-engine-export.md` | Export targets derived from ecosystem analysis |
| `07-CROSS-ENGINE.md` | CnCNet bridge architecture |
| `cross-engine/relay-security.md` | CnCNet trust tier classification |
| `cross-engine/compatibility-packs.md` | `classic-ea` balance for CnCNet compatibility |
| `modding/workshop.md` | One-click upload pattern from OpenRA |
