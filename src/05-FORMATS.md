# 05 — File Formats & Original Source Insights

## Formats to Support (ra-formats crate)

### Binary Formats (from original game / OpenRA)

| Format | Purpose           | Notes                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------ | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `.mix` | Archive container | Flat archive with CRC-based filename hashing (rotate-left-1 + add), 6-byte `FileHeader` + sorted `SubBlock` index (12 bytes each). Extended format adds Blowfish encryption + SHA-1 digest. No per-file compression. See § MIX Archive Format for full struct definitions                                                                                                                                          |
| `.shp` | Sprite sheets     | Frame-based, palette-indexed (256 colors). `ShapeBlock_Type` container with per-frame `Shape_Type` headers. LCW-compressed frame data (or uncompressed via `NOCOMP` flag). Supports compact 16-color mode, horizontal/vertical flip, scaling, fading, shadow, ghost, and predator draw modes                                                                                                                       |
| `.tmp` | Terrain tiles     | IFF-format icon sets — collections of 24×24 palette-indexed tiles. Chunks: ICON/SINF/SSET/TRNS/MAP/RPAL/RTBL. SSET data may be LCW-compressed. RA version adds `MapWidth`/`MapHeight`/`ColorMap` for land type lookup. TD and RA `IControl_Type` structs differ — see § TMP Terrain Tile Format                                                                                                                    |
| `.pal` | Color palettes    | Raw 768 bytes (256 × RGB), no header. Components in 6-bit VGA range (0–63), not 8-bit. Convert to 8-bit via left-shift by 2. Multiple palettes per scenario (temperate, snow, interior, etc.)                                                                                                                                                                                                                      |
| `.aud` | Audio             | Westwood IMA ADPCM compressed. 12-byte `AUDHeaderType`: sample rate (Hz), compressed/uncompressed sizes, flags (stereo/16-bit), compression ID. Codec uses dual 1424-entry lookup tables (`IndexTable`/`DiffTable`) for 4-bit-nibble decoding. Read + write: Asset Studio (D040) converts .aud ↔ .wav/.ogg so modders can extract original sounds for remixing and convert custom recordings to classic AUD format |
| `.vqa` | Video             | VQ vector quantization cutscenes. Chunk-based IFF structure (WVQA/VQHD/FINF/VQFR/VQFK). Codebook blocks (4×2 or 4×4 pixels), LCW-compressed frames, interleaved audio (PCM/Westwood ADPCM/IMA ADPCM). Read + write: Asset Studio (D040) converts .vqa ↔ .mp4/.webm for campaign creators                                                                                                                           |

### Remastered Collection Formats (Petroglyph)

HD asset formats from the C&C Remastered Collection (EA, 2020). Format definitions derived from the GPL v3 C++ DLL source and community documentation. See [D075](decisions/09c/D075-remastered-format-compat.md) for full import pipeline and legal model.

| Format       | Purpose           | Notes                                                                                                                                                                                                                                                                |
| ------------ | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.meg`       | Archive container | Petroglyph archive format (from Empire at War lineage). Header + file table + packed data. Read-only in `ra-formats`. Community tools: OS Big Editor, OpenSage.                                                                                                      |
| `.tga+.meta` | HD sprite sheets  | 32-bit RGBA TGA "megasheets" — all frames of a unit/building composited into one large atlas. Paired `.meta` JSON file provides per-frame geometry: `{"size":[w,h],"crop":[x,y,w,h]}`. Player colors use chroma-key green (HSV hue ~110) instead of palette indices. |
| `.dds`       | GPU textures      | DirectDraw Surface (BC1/BC3/BC7). Terrain, UI chrome, effects. Convert to KTX2 or PNG at import time.                                                                                                                                                                |
| `.bk2`       | HD video (Bink2)  | Proprietary RAD Game Tools codec. Cutscenes and briefings. Converted to WebM (VP9) at import time — IC does not ship a Bink2 runtime decoder.                                                                                                                        |
| `.wav` (HD)  | Remixed audio     | Standard WAV containers (Microsoft ADPCM). Plays natively in IC's Kira audio pipeline. No conversion needed.                                                                                                                                                         |
| `.pgm`       | Map package       | MEG file with different extension. Contains map + preview image + metadata. Reuse `MegArchive` parser.                                                                                                                                                               |

### Text Formats

| Format            | Purpose                     | Notes                                              |
| ----------------- | --------------------------- | -------------------------------------------------- |
| `.ini` (original) | Game rules                  | Original Red Alert format                          |
| MiniYAML (OpenRA) | Game rules, maps, manifests | Custom dialect, needs converter                    |
| YAML (ours)       | Game rules, maps, manifests | Standard spec-compliant YAML                       |
| `.oramap`         | OpenRA map package          | ZIP archive containing map.yaml + terrain + actors |

### Canonical Asset Format Recommendations (D049)

New Workshop content should use **Bevy-native modern formats** by default. C&C legacy formats are fully supported for backward compatibility but are not the recommended distribution format. The engine loads both families at runtime — no manual conversion is ever required.

| Asset Type      | Recommended (new content)      | Legacy (existing)      | Why Recommended                                                                                                     |
| --------------- | ------------------------------ | ---------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Music**       | OGG Vorbis (128–320kbps)       | .aud (ra-formats)      | Bevy default feature, stereo 44.1kHz, ~1.4MB/min. Open, patent-free, WASM-safe, security-audited by browser vendors |
| **SFX**         | WAV (16-bit PCM) or OGG        | .aud (ra-formats)      | WAV = zero decode latency for gameplay-critical sounds. OGG for larger ambient sounds                               |
| **Voice**       | OGG Vorbis (96–128kbps)        | .aud (ra-formats)      | Transparent quality for speech. 200+ EVA lines stay under 30MB                                                      |
| **Sprites**     | PNG (RGBA or indexed)          | .shp+.pal (ra-formats) | Bevy-native via `image` crate. Lossless, universal tooling. Palette-indexed PNG preserves classic aesthetic         |
| **HD Textures** | KTX2 (BC7/ASTC GPU-compressed) | N/A                    | Zero-cost GPU upload, Bevy-native. `ic mod build` can batch-convert PNG→KTX2                                        |
| **Terrain**     | PNG tiles                      | .tmp+.pal (ra-formats) | Same as sprites — theater tilesets are sprite sheets                                                                |
| **Cutscenes**   | WebM (VP9, 720p–1080p)         | .vqa (ra-formats)      | Open, royalty-free, browser-compatible (WASM), ~5MB/min at 720p                                                     |
| **3D Models**   | GLTF/GLB                       | N/A                    | Bevy's native 3D format                                                                                             |
| **Palettes**    | .pal (768 bytes)               | .pal (ra-formats)      | Already tiny and universal in the C&C community — no change needed                                                  |
| **Maps**        | IC YAML                        | .oramap (ZIP+MiniYAML) | Already designed (D025, D026)                                                                                       |

**Why modern formats:** (1) Bevy loads them natively — zero custom code, full hot-reload and async loading. (2) Security — OGG/PNG parsers are fuzz-tested and browser-audited; our custom .aud/.shp parsers are not. (3) Multi-game — non-C&C game modules (D039) won't use .shp or .aud. (4) Tooling — every editor exports PNG/OGG/WAV/WebM; nobody's toolchain outputs .aud. (5) WASM — modern formats work in browser builds out of the box.

The Asset Studio (D040) converts in both directions. See `decisions/09e/D049-workshop-assets.md` for full rationale, storage comparisons, and distribution strategy.

### ra-formats Crate Goals

1. Parse all above formats reliably
2. Extensive tests against known-good OpenRA data
3. `miniyaml2yaml` converter tool
4. CLI tool to dump/inspect/validate RA assets
5. **Write support (Phase 6a):** .shp generation from frames (LCW compression + frame offset tables), .pal writing (trivial — 768 bytes), .aud encoding (IMA ADPCM compression from PCM input), .vqa encoding (VQ codebook generation + frame differencing + audio interleaving), optional .mix packing (CRC hash table generation) — required by Asset Studio (D040). All encoders reference the EA GPL source code implementations directly (see § Binary Format Codec Reference)
6. Useful as standalone crate (builds project credibility)
7. Released open source early (Phase 0 deliverable, read-only; write support added Phase 6a)

### Non-C&C Format Landscape

The `ra-formats` crate covers the C&C format family, but the engine (D039) supports non-C&C games via the `FormatRegistry` and WASM format loaders (see `04-MODDING.md` § WASM Format Loader API Surface). Analysis of six major OpenRA community mods (see `research/openra-mod-architecture-analysis.md`) reveals the scope of formats that non-C&C total conversions require:

| Game (Mod)             | Custom Formats Required                                                   | Notes                                                     |
| ---------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------- |
| KKnD (OpenKrush)       | `.blit`, `.mobd`, `.mapd`, `.lvl`, `.son`, `.soun`, `.vbc` (15+ decoders) | Entirely proprietary format family; zero overlap with C&C |
| Dune II (d2)           | `.icn`, `.cps`, `.wsa`, `.shp` variant, `.adl`, custom map format (6+)    | Different `.shp` than C&C; incompatible parsers           |
| Swarm Assault (OpenSA) | Custom creature sprites, terrain tiles                                    | Format details vary by content source                     |
| Tiberian Dawn HD       | MegV3 archives, 128×128 HD tiles (`RemasterSpriteSequence`)               | Different archive format than `.mix`                      |
| OpenHV                 | None — uses PNG/WAV/OGG exclusively                                       | Original game content avoids legacy formats entirely      |

**Key insight:** Non-C&C games on the engine need 0–15+ custom format decoders, and there is zero format overlap with C&C. This validates the `FormatRegistry` design — the engine cannot hardcode any format assumption. `ra-formats` is one format loader plugin among potentially many.

**Cross-engine validation:** Godot's `ResourceFormatLoader` follows the same pattern — a pluggable interface where any module registers format handlers (recognized extensions, type specializations, caching modes) and the engine dispatches to the correct loader at runtime. Godot's implementation includes threaded loading, load caching (reuse/ignore/replace), and recursive dependency resolution for complex assets. IC's `FormatRegistry` via Bevy's asset system should support the same capabilities: threaded background loading, per-format caching policy, and declared dependencies between assets (e.g., a sprite sheet depends on a palette). See `research/godot-o3de-engine-analysis.md` § Asset Pipeline.

### Content Source Detection

Games use different distribution platforms, and each stores assets in different locations. Analysis of TiberianDawnHD (see `research/openra-mod-architecture-analysis.md`) shows a robust pattern for detecting installed game content:

```rust
/// Content sources — where game assets are installed.
/// Each game module defines which sources it supports.
pub enum ContentSource {
    Steam { app_id: u32 },           // e.g., Steam AppId 2229870 (TD Remastered)
    Origin { registry_key: String }, // Windows registry path to install dir
    Gog { game_id: String },         // GOG Galaxy game identifier
    Directory { path: PathBuf },     // Manual install / disc copy
}
```

TiberianDawnHD detects Steam via AppId, Origin via Windows registry key, and GOG via standard install paths. IC should implement a `ContentDetector` that probes all known sources for each supported game and presents the user with detected installations at first run. This handles the critical UX question "where are your game assets?" without requiring manual path entry — the same approach used by OpenRA, CorsixTH, and other reimplementation projects.

**Phase:** Content detection ships in Phase 0 as part of `ra-formats` (for C&C assets). Game module content detection in Phase 1.

### Browser Asset Storage

The `ContentDetector` pattern above assumes filesystem access — probing Steam, Origin, GOG, and directory paths. None of this works in a browser build (WASM target). Browsers have no access to the user's real filesystem. IC needs a dedicated browser asset storage strategy.

**Browser storage APIs** (in order of preference):

- **OPFS (Origin Private File System):** The newest browser storage API (~2023). Provides a real private filesystem with file/directory operations and synchronous access from Web Workers. Best performance for large binary assets like `.mix` archives. Primary storage backend for IC's browser build.
- **IndexedDB:** Async NoSQL database. Stores structured data and binary blobs. Typically 50MB–several GB (browser-dependent, user-prompted above quota). Wider browser support than OPFS. Fallback storage backend.
- **localStorage:** Simple key-value string store, ~5-10MB limit, synchronous. Too small for game assets — suitable only for user preferences and settings.

**Storage abstraction:**

```rust
/// Platform-agnostic asset storage.
/// Native builds use the filesystem directly. Browser builds use OPFS/IndexedDB.
pub trait AssetStore: Send + Sync {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>>;
    fn write(&self, path: &VirtualPath, data: &[u8]) -> Result<()>;
    fn exists(&self, path: &VirtualPath) -> bool;
    fn list_dir(&self, path: &VirtualPath) -> Result<Vec<VirtualPath>>;
    fn delete(&self, path: &VirtualPath) -> Result<()>;
    fn available_space(&self) -> Result<u64>; // quota management
}

pub struct NativeStore { root: PathBuf }
pub struct BrowserStore { /* OPFS primary, IndexedDB fallback */ }
```

**Browser first-run asset acquisition:**

1. User opens IC in a browser tab. No game assets exist in browser storage yet.
2. First-run wizard presents options: (a) drag-and-drop `.mix` files from a local RA installation, (b) paste a directory path to bulk-import, or (c) download a free content pack if legally available (e.g., freeware TD/RA releases).
3. Imported files are stored in the OPFS virtual filesystem under a structured directory (similar to Chrono Divide's `📁 /` layout: game archives at root, mods in `mods/<modId>/`, maps in `maps/`, replays in `replays/`).
4. Subsequent launches skip import — assets persist in OPFS across sessions.

**Browser mod installation:**

Mods are downloaded as archives (via Workshop HTTP API or direct URL), extracted in-browser (using a JS/WASM decompression library), and written to `mods/<modId>/` in the virtual filesystem. The in-game mod browser triggers download and extraction. Lobby auto-download (D030) works identically — the `AssetStore` trait abstracts the actual storage backend.

**Storage quota management:**

Browsers impose per-origin storage limits (typically 1-20GB depending on browser and available disk). IC's browser build should: (a) check `available_space()` before large downloads, (b) surface clear warnings when approaching quota, (c) provide a storage management UI (like Chrono Divide's "Options → Storage") showing per-mod and per-asset space usage, (d) allow selective deletion of cached assets.

**Bevy integration:** Bevy's asset system already supports custom asset sources. The `BrowserStore` registers as a Bevy `AssetSource` so that `asset_server.load("ra2.mix")` transparently reads from OPFS on browser builds and from the filesystem on native builds. No game code changes required — the abstraction lives below Bevy's asset layer.

**Phase:** `AssetStore` trait and `BrowserStore` implementation ship in Phase 7 (browser build). The trait definition should exist from Phase 0 so that `NativeStore` is used consistently — this prevents filesystem assumptions from leaking into game code. Chrono Divide's browser storage architecture (OPFS + IndexedDB, virtual directory structure, mod folder isolation) validates this approach.


---

## Sub-Pages

| Section                     | Topic                                                                                              | File                                                                 |
| --------------------------- | -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Binary Codecs               | MIX, SHP, LCW, TMP, PAL, AUD, VQA codec specs + EA source insights + coordinate system translation | [binary-codecs.md](formats/binary-codecs.md)                         |
| Save & Replay Formats       | Save game format (.icsave) + replay file format (.icrep)                                           | [save-replay-formats.md](formats/save-replay-formats.md)             |
| — Keyframes & Analysis      | Keyframe snapshot types, delta structs, seeking algorithm, analysis event taxonomy                 | [replay-keyframes-analysis.md](formats/replay-keyframes-analysis.md) |
| Backup, Screenshot & Import | Backup archive format (D061) + screenshot format + owned-source import/extraction pipeline         | [backup-screenshot-import.md](formats/backup-screenshot-import.md)   |
