# Data Layout Spectrum & Infrastructure Performance

> **Sub-page of:** [Performance Philosophy](../10-PERFORMANCE.md)
> **Status:** Design guidance. Applies from Phase 0 onward for format decisions; runtime optimizations phased per subsystem.

## Overview

IC's Efficiency Pyramid (algorithm → cache → LOD → amortize → zero-alloc → parallelism) applies primarily to the simulation hot path. But significant computation also occurs in non-ECS subsystems: P2P distribution, Workshop indexing, fog-of-war bitmaps, AI influence maps, damage resolution, and replay analysis.

These subsystems don't live in Bevy's ECS and therefore don't automatically benefit from ECS data layout. This page defines the data layout spectrum and maps each subsystem to its optimal layout.

## The Data Layout Spectrum

From most flexible to most hardware-efficient:

| Layout | Description | Best For | Rust Crate |
|---|---|---|---|
| **Full ECS** | Bevy archetype tables | Sim entities (units, buildings, projectiles) | `bevy_ecs` |
| **SoA** | Struct-of-Arrays via derive macro | Index/catalog data with frequent column scans | `soa-rs` |
| **AoSoA** | Array-of-Structs-of-Arrays, tiles of 4–8 | Batch processing with mixed field access | Manual or `nalgebra`+`simba` |
| **Arrow** | Columnar format, zero-copy from disk | Analytics, replay analysis, read-heavy data | `arrow-rs` / `minarrow` |
| **SIMD bitfields** | Wide register operations on packed bits | Boolean operations (fog, piece tracking) | `wide` |

Each step trades flexibility for raw throughput. The choice depends on access pattern, data volume, and update frequency.

## Per-Subsystem Mapping

| Subsystem | Current (Design) | Recommended Layout | Rationale |
|---|---|---|---|
| **Sim entities** | Full ECS (Bevy) | Keep ECS | Bevy's archetype SoA is already optimal |
| **Workshop resource index** | `Vec<ResourceListing>` | **SoA** via `soa-rs` | Filter by category scans 10KB contiguous array vs. multi-MB scattered structs |
| **P2P piece bitfields** | Not yet designed | **SIMD bitfields** (`wide`) | Piece availability is boolean algebra; 256 pieces per SIMD instruction |
| **Fog-of-war bitmap** | Per-player grid | **SIMD row bitmaps** | Reveal = single SIMD OR per row; 128 cells per register |
| **Influence maps** | `[i32; MAP_AREA]` | **`#[repr(C, align(64))]`** | Auto-vectorized decay/normalize with AVX2/AVX-512 |
| **Damage event buffer** | `Vec<DamageEvent>` | **AoSoA tiles of 8** | 8 damage events processed per SIMD pass |
| **Replay analysis** | Not yet designed | **Arrow columnar** | SQL-like queries over order streams; zero-copy from disk |
| **VersusTable** | Flat array | Keep current | Already cache-friendly; no change needed |
| **Peer scoring** | Not yet designed | **SoA** | Column scans for peer ranking (sort by speed, filter by availability) |

## Key Recommendations

### Workshop Resource Index — SoA

Store browse-time-hot fields in SoA layout:

```rust
use soa_rs::Soars;

#[derive(Soars)]
struct ResourceIndex {
    id: InternedResourceId,        // 4 bytes
    category: ResourceCategory,     // 1 byte
    game_module: GameModuleId,      // 1 byte
    platform_bits: u8,              // bitflags
    rating_centile: u8,             // 0-100, pre-computed
    download_count: u32,            // sort-by-popularity
    name_hash: u32,                 // fast text search pre-filter
}
```

Filtering 10,000 resources by category scans a contiguous 10KB `[ResourceCategory; 10000]` (L1 cache) instead of touching scattered multi-KB structs. Full resource details (description, tags, author info) stored in a separate cold-tier `Vec<ResourceDetail>` indexed by position.

### P2P Piece Tracking — SIMD Bitfields

```rust
use wide::u64x4;

struct PieceBitfield {
    blocks: Vec<u64x4>,  // 256 bits per block
}

impl PieceBitfield {
    fn useful_pieces(&self, peer_have: &PieceBitfield, in_flight: &PieceBitfield) -> PieceBitfield {
        PieceBitfield {
            blocks: self.blocks.iter()
                .zip(peer_have.blocks.iter())
                .zip(in_flight.blocks.iter())
                .map(|((mine, theirs), flying)| {
                    let need = !*mine;
                    need & *theirs & !*flying
                })
                .collect()
        }
    }
}
```

For a 50MB mod with 200 pieces, the entire useful-piece computation is one loop iteration. For 2000 pieces, it's 8 iterations — versus 2000 scalar boolean operations.

### Fog-of-War Bitmap — SIMD Rows

```rust
use wide::u64x2;

struct FogBitmap {
    rows: Vec<u64x2>,  // One entry per map row; 128 cells per register
}

impl FogBitmap {
    fn reveal(&mut self, cx: usize, cy: usize, sight_mask: &SightMask) {
        for (dy, mask_row) in sight_mask.rows.iter().enumerate() {
            let y = cy + dy - sight_mask.radius;
            if y < self.rows.len() {
                self.rows[y] |= shift_mask(*mask_row, cx, sight_mask.radius);
            }
        }
    }
}
```

For a unit with sight radius 5, revealing visibility touches ~10 map rows — 10 SIMD ORs instead of ~300 scalar bit-sets.

### Influence Maps — Aligned Arrays

```rust
#[repr(C, align(64))]
struct InfluenceMap {
    cells: [i32; 128 * 128],  // 64KB, fits in L2
}

impl InfluenceMap {
    fn decay(&mut self, numerator: i32, denominator: i32) {
        for cell in self.cells.iter_mut() {
            *cell = (*cell * numerator) / denominator;
        }
    }
}
```

With 64-byte alignment and simple loop body, LLVM auto-vectorizes: 128×128 decay in ~1000 SIMD instructions (AVX2) vs. ~16,384 scalar.

### Damage Events — AoSoA Tiles

```rust
#[repr(C, align(32))]
struct DamageEventTile {
    attacker_weapon: [InternedId; 8],
    defender_armor:  [InternedId; 8],
    base_damage:     [i32; 8],
    distance_sq:     [i32; 8],
    attacker_entity: [Entity; 8],
    defender_entity: [Entity; 8],
}
```

VersusTable lookup (weapon × armor → modifier) can be batched: gather 8 indices, look up 8 modifiers, multiply 8 damages — auto-vectorized.

### Replay Analysis — Arrow Columnar

> **Not a storage format.** The canonical replay storage format is `.icrep` (see `formats/save-replay-formats.md`). Arrow is a **derived analytics/index layer** — replay order streams are converted to Arrow `RecordBatch` for analysis queries, not stored as Arrow on disk. The `.icrep` file remains the source of truth; Arrow representation is transient and computed on demand.

When the replay analysis system is built (Phase 4–5), convert replay order streams to Arrow format for querying. Each replay becomes a `RecordBatch` with columns for tick, player_id, order_type, target coordinates, and payloads. Arrow's compute kernels provide SIMD-vectorized filter, sort, and aggregate operations. Zero-copy from disk — no deserialization needed for the columnar representation once built.

## Efficiency Pyramid Applied to P2P/Workshop

The simulation Efficiency Pyramid applies to infrastructure subsystems too:

| Layer | Sim Example | Infrastructure Equivalent |
|---|---|---|
| **Algorithm** | JPS+ pathfinding | Content-aware `.icpkg` piece ordering (manifest first, sprites before audio) |
| **Cache-friendly** | Archetype SoA | SoA Workshop index, hot/warm/cold cache tiers |
| **LOD** | Sim LOD per distance band | Adaptive `PeerLOD` — full attention for active transfers, background for seeds |
| **Amortize** | Fog updates every N ticks | Staggered background ops (`tick % N` scheduling for subscription checks, cache cleanup) |
| **Zero-alloc** | `TickScratch` pre-allocated | `InternedResourceId` (4-byte interned vs. variable-length string), scratch buffers for piece assembly |
| **Parallelism** | `par_iter()` last resort | Pipelined piece validation (hash in background while downloading next piece) |

## Content-Aware Piece Ordering

The `.icpkg` package format should define canonical file ordering:

1. Package manifest (metadata, dependency list)
2. Thumbnail / preview image
3. YAML rules (small, needed for lobby display)
4. Sprite sheets (needed for rendering)
5. Audio files (can stream, lowest priority)

This ordering means the first pieces of a download contain the metadata and preview — enough to display in the Workshop browser before the full package is downloaded.

## Cache Tiering

```
Hot tier:   mmap'd — actively playing/editing content (instant access)
Warm tier:  on-disk — recently used, subscribed, prefetched (disk seek)
Cold tier:  evictable — LRU, unsubscribed (may need re-download)
```

The tier management runs on the stagger schedule (not every tick), promoting content on access and demoting on LRU eviction.

## Implementation Phasing

| Phase | What to Decide/Implement |
|---|---|
| **Phase 0–1** | Define `.icpkg` piece ordering; add `#[repr(C, align(64))]` to influence map types; design `ResourceIndex` SoA layout; design P2P piece bitfield type |
| **Phase 2** | Implement fog SIMD bitmaps; implement influence map alignment; design AoSoA damage tiles |
| **Phase 3–4** | Implement SoA Workshop index; implement stagger schedule; implement cache tiering |
| **Phase 4–5** | Design replay analysis on Arrow; implement piece validation pipeline |
| **Phase 5+** | Profile and implement AoSoA damage tiles (only if bottleneck shown); Arrow replay analysis |

## Dependency Summary

| Crate | License | Use | WASM |
|---|---|---|---|
| `soa-rs` | MIT/Apache-2.0 | SoA layout for Workshop index, peer tables | Yes |
| `wide` | Zlib/Apache-2.0/MIT | SIMD bitfields, batch arithmetic | Partial (scalar fallback) |
| `arrow-rs` | Apache-2.0 | Replay analysis, analytics (Phase 4–5) | Yes |
| `datafusion` | Apache-2.0 | SQL queries over replay data (optional) | No |

All compatible with IC's GPL v3 + modding exception license.
