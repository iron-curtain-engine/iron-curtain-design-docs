# EA Editor Source Study — cnc-formats Parsing Gaps

> **Status:** Research document. Informs `cnc-formats` (D076) and `ic-cnc-content` development.
> **Date:** 2026-02

## 1. Executive Summary

Study of EA's GPL-released FinalSun/FA2 source code and community documentation (ModEnc wiki, XCC Utilities) reveals 9 specific parsing gaps in the `cnc-formats` crate's TS/RA2 map handling. The crate currently provides typed accessors for INI-style map sections but does not decode their binary or structured content. This document catalogs each gap with the data structures, dependencies, and implementation priorities needed to close them.

**Clean-room compliance:** All implementations must derive from ModEnc wiki documentation and XCC Utilities specifications — NOT from the EA GPL source code directly (to keep `cnc-formats` under MIT/Apache-2.0). The EA source is studied for *understanding the data model*, not for code copying.

## 2. Current State of cnc-formats

The `cnc-formats` crate (`map_ra2` module) correctly:
- Parses RA2/YR `.map` files as INI with named sections
- Provides typed accessor methods for section keys
- Handles coordinate packing/unpacking for cell references
- Supports read-only map operations

What it does NOT do:
- Decode binary-packed sections (`IsoMapPack5`, `OverlayPack`, `OverlayDataPack`, `PreviewPack`)
- Parse structured entity placements (`[Structures]`, `[Infantry]`, `[Units]`, `[Aircraft]`)
- Decode trigger/scripting sections (`[Triggers]`, `[Events]`, `[Actions]`, etc.)
- Provide typed tunnel tube or waypoint data

## 3. Gap Catalog

### Gap 1: IsoMapPack5 Binary Terrain Decoding

**What:** The `[IsoMapPack5]` section contains Base64-encoded, LZO-compressed binary terrain data. Each decoded cell is an 11-byte `MAPFIELDDATA` struct:

```rust
/// 11 bytes per cell, packed little-endian
pub struct IsoMapCell {
    pub x: u16,           // Cell X coordinate
    pub y: u16,           // Cell Y coordinate
    pub tile_index: u16,  // Tile type index into the tileset
    pub sub_tile: u8,     // Sub-tile within the tile (for multi-cell tiles)
    pub height: u8,       // Cell height level (0–14 typically)
    pub ice_growth: u8,   // Ice growth stage (RA2 mechanic)
    pub _reserved: [u8; 2], // Pad/reserved bytes
}
```

**Decoding pipeline:** Base64 → LZO decompress → read 11-byte structs until end of buffer

**LZO dependency:** Use `lzo1x-1` crate (MIT license — compatible with cnc-formats' MIT/Apache-2.0). The compression is specifically LZO1X-1 (same as used by Westwood throughout the TS/RA2 engine).

**Proposed API:**
```rust
impl MapRA2 {
    pub fn decode_iso_map_pack(&self) -> Result<Vec<IsoMapCell>, FormatError>;
}
```

**Priority:** High. Terrain is the foundation of any map — without this, maps are visually blank.

### Gap 2: OverlayPack & OverlayDataPack Decoding

**What:** `[OverlayPack]` contains overlay type indices (ore, gems, walls, bridges). `[OverlayDataPack]` contains overlay metadata (health, frame index for animation). Both are Base64-encoded, LZO-compressed arrays.

```rust
pub struct OverlayData {
    /// 262144 entries (512×512 max map), one byte per cell
    pub overlay_types: Vec<u8>,   // 0xFF = no overlay
    /// Same size, one byte per cell
    pub overlay_frames: Vec<u8>,  // sub-frame / health / stage
}
```

**Proposed API:**
```rust
impl MapRA2 {
    pub fn decode_overlay_pack(&self) -> Result<OverlayData, FormatError>;
}
```

**Priority:** High. Ore fields and gems are critical for gameplay; bridges and walls affect pathing.

### Gap 3: Entity Placement Sections

**What:** Six sections define entity placement with different field counts:

**[Structures]** — 17 comma-separated fields per entry:
```
ID=Owner,TypeID,Health,X,Y,Facing,Tag,AISellable,AIRebuildable,Powered,Upgrades,Spotlight,Upgrade1,Upgrade2,Upgrade3,AIRepairable,Nominal
```

**[Infantry]** — 14 fields:
```
ID=Owner,TypeID,Health,X,Y,SubCell,Mission,Facing,Tag,Veteran,Group,OnBridge,AutocreateNoRecruitable,AutocreateYesRecruitable
```

**[Units]** — 14 fields:
```
ID=Owner,TypeID,Health,X,Y,Facing,Mission,Tag,Veteran,Group,OnBridge,FollowsIndex,AutocreateNoRecruitable,AutocreateYesRecruitable
```

**[Aircraft]** — 12 fields per entry

**[Terrain]** — Terrain objects (trees, rocks): `Cell=TypeID`

**[Smudge]** — Crater/scorch marks: `TypeID,X,Y,Ignored`

**Proposed types:**
```rust
pub struct Structure {
    pub id: u32,
    pub owner: String,
    pub type_id: String,
    pub health: u16,       // 0–256 (percentage × 256/100)
    pub x: u16,
    pub y: u16,
    pub facing: u8,        // 0–255
    pub tag: String,       // trigger tag (empty = none)
    pub upgrades: u8,      // number of upgrades applied
    pub upgrade_slots: [String; 3],
    // ... remaining fields
}

pub struct Infantry {
    pub id: u32,
    pub owner: String,
    pub type_id: String,
    pub health: u16,
    pub x: u16,
    pub y: u16,
    pub sub_cell: u8,      // 0–4 (position within cell)
    pub mission: String,
    pub facing: u8,
    pub tag: String,
    pub veteran_level: u8,
    // ... remaining fields
}
```

**Proposed API:**
```rust
impl MapRA2 {
    pub fn structures(&self) -> Result<Vec<Structure>, FormatError>;
    pub fn infantry(&self) -> Result<Vec<Infantry>, FormatError>;
    pub fn units(&self) -> Result<Vec<Unit>, FormatError>;
    pub fn aircraft(&self) -> Result<Vec<Aircraft>, FormatError>;
    pub fn terrain_objects(&self) -> Result<Vec<TerrainObject>, FormatError>;
    pub fn smudges(&self) -> Result<Vec<Smudge>, FormatError>;
}
```

**Priority:** High. Entity placement is required for any meaningful map rendering or analysis.

### Gap 4: Trigger & Scripting Sections

**What:** The TS/RA2 trigger system uses 8 interlinked INI sections:

- **[Triggers]:** `ID=Owner,TriggerName,EventListID,ActionListID,Repair,Loop,Disabled,...`
- **[Events]:** `EventListID=EventCount,EventType1,Param1,Param2,...,EventType2,...`
- **[Actions]:** `ActionListID=ActionCount,ActionType1,Param1,...,Waypoint,...`
- **[Tags]:** `TagID=Repeat,Name,TriggerID` (links triggers to map elements)
- **[CellTags]:** `PackedCell=TagID` (attaches tags to map cells)
- **[TaskForces]:** `ID=Name,Group,UnitCount1,UnitType1,...`
- **[ScriptTypes]:** `ID=Name,ActionCount,ActionType1,Param1,...`
- **[TeamTypes]:** `ID=Name,Owner,Priority,Max,TechLevel,TaskForce,Script,Tag,...`
- **[AITriggerTypes]:** Links AI behavior to trigger conditions

**Proposed types:**
```rust
pub struct Trigger {
    pub id: String,
    pub owner: String,
    pub name: String,
    pub event_list_id: String,
    pub action_list_id: String,
    pub repeating: bool,
    pub disabled: bool,
}

pub struct TriggerEvent {
    pub event_type: u32,
    pub params: Vec<String>,   // parameter interpretation depends on event_type
}

pub struct TriggerAction {
    pub action_type: u32,
    pub params: Vec<String>,   // parameter interpretation depends on action_type
    pub waypoint: Option<u32>,
}

pub struct Tag {
    pub id: String,
    pub repeat_type: u8,   // 0=once, 1=repeat, 2=repeat-and
    pub name: String,
    pub trigger_id: String,
}

pub struct TeamType {
    pub id: String,
    pub name: String,
    pub owner: String,
    pub task_force_id: String,
    pub script_id: String,
    pub tag: String,
    pub priority: u32,
    pub max: u32,
    // ... 20+ additional fields
}
```

**Priority:** Medium. Essential for scenario editing and map analysis, but maps can be rendered without triggers.

### Gap 5: Tunnel Tube Data

**What:** `[Tubes]` section defines underground tunnel connections between cells:
```
StartX,StartY,Facing,EndX,EndY,Facing,DirectionList
```
Where `DirectionList` is a sequence of direction bytes (0–7) encoding the underground path.

**Proposed API:**
```rust
pub struct TunnelTube {
    pub start: CellPos,
    pub start_facing: u8,
    pub end: CellPos,
    pub end_facing: u8,
    pub path: Vec<u8>,   // direction bytes along the tube
}

impl MapRA2 {
    pub fn tunnel_tubes(&self) -> Result<Vec<TunnelTube>, FormatError>;
}
```

**Priority:** Medium. Tunnel tubes affect pathfinding and are a key TS/RA2 feature.

### Gap 6: Waypoint Coordinate Decoding

**What:** `[Waypoints]` section contains packed coordinates:
```
WaypointNumber=PackedCell
```
Where `PackedCell = Y * 1000 + X` (standard TS/RA2 cell packing).

The crate may already have the packing/unpacking utility, but there's no typed waypoint collection.

**Proposed API:**
```rust
pub struct Waypoint {
    pub number: u32,
    pub cell: CellPos,
}

impl MapRA2 {
    pub fn waypoints(&self) -> Result<Vec<Waypoint>, FormatError>;
}
```

**Priority:** High (low effort). Waypoints define spawn positions, objectives, and trigger locations.

### Gap 7: Preview Image Decoding

**What:** `[PreviewPack]` contains a Base64-encoded, LZO-compressed bitmap preview of the map (shown in game's map selection screen). Format is raw RGB24 or similar.

**Proposed API:**
```rust
impl MapRA2 {
    pub fn decode_preview(&self) -> Result<PreviewImage, FormatError>;
}
```

**Priority:** Low. Nice for map browsers and asset studio but not critical for gameplay.

### Gap 8: House & Country Definitions

**What:** `[Houses]` and `[Countries]` sections define the players/factions in the map with their starting conditions (credits, allies, color, country type, player control type).

```rust
pub struct House {
    pub name: String,
    pub iq: u32,
    pub edge: String,        // spawn edge
    pub allies: Vec<String>,
    pub color: String,
    pub country: String,
    pub credits: u32,
    pub player_control: String,  // "human" or "computer"
    pub tech_level: u32,
}
```

**Priority:** Medium. Required for correct multi-player map loading and AI configuration.

### Gap 9: FIELDDATA Unified Per-Cell Model

**What:** A unified per-cell data model that combines terrain (Gap 1), overlays (Gap 2), cell tags (Gap 4), and derived properties into a single queryable grid. This belongs in `ic-cnc-content` (not cnc-formats) because it involves game-engine-specific interpretation.

```rust
/// Unified cell data — lives in ic-cnc-content, not cnc-formats
pub struct FieldData {
    pub terrain: IsoMapCell,
    pub overlay_type: Option<u8>,
    pub overlay_frame: Option<u8>,
    pub cell_tag: Option<String>,
    pub height: u8,
    pub passable: bool,        // derived from terrain + overlay
    pub buildable: bool,       // derived
}
```

**Priority:** Medium-Low. This is a convenience layer on top of the raw data.

## 4. Implementation Priority & Effort

| Priority | Gaps | Effort | Phase Target |
|---|---|---|---|
| **High** | Gap 1 (terrain), Gap 2 (overlays), Gap 3 (entities), Gap 6 (waypoints) | ~3 weeks | Phase 0–1 |
| **Medium** | Gap 4 (triggers), Gap 5 (tunnels), Gap 8 (houses) | ~4 weeks | Phase 2 |
| **Low** | Gap 7 (preview), Gap 9 (FIELDDATA) | ~1.5 weeks | Phase 2–3 |

Total: ~8.5 weeks of focused implementation.

## 5. Dependency Requirements

| Dependency | License | Purpose |
|---|---|---|
| `lzo1x-1` | MIT | LZO1X-1 decompression for IsoMapPack5, OverlayPack, PreviewPack |
| `base64` | MIT/Apache-2.0 | Already a dependency of cnc-formats |

The LZO dependency is the only new addition. It's MIT-licensed, compatible with cnc-formats' MIT/Apache-2.0 dual license.

## 6. Clean-Room Compliance

All type definitions and decoding logic must be derived from:
- **ModEnc wiki** (modenc.renegadeprojects.com) — community-maintained format documentation
- **XCC Utilities** source and documentation — Olaf van der Spek's format reverse-engineering
- **Empirical testing** — hex-dump analysis of known maps

The EA GPL source code is studied for *understanding what the data means* (field semantics, edge cases), not for code structure or algorithms. This distinction preserves cnc-formats' MIT/Apache-2.0 license per D076.

## 7. Relationship to Design Documents

| Document | Relevance |
|---|---|
| `decisions/09a/D076-standalone-crates.md` | cnc-formats feature flag roadmap; new gaps need Phase 1–2 entries |
| `05-FORMATS.md` / `formats/binary-codecs.md` | Binary format documentation |
| `decisions/09f/D038-scenario-editor.md` | Editor requires all 9 gaps for full TS/RA2 map editing |
| `decisions/09c/D075-remastered-format-compat.md` | Remastered Collection format overlap |
