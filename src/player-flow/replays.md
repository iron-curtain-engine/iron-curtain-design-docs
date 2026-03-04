## Replays

> **Cross-game analysis:** See `research/replay-playback-ux-survey.md` for the detailed source study covering SC2, AoE2:DE/CaptureAge, Dota 2, CS2, CoH3, WC3:Reforged, LoL, and Fortnite that informed this spec.

---

### Replay Browser

```
Main Menu → Replays
```

```
┌──────────────────────────────────────────────────────────────────┐
│  REPLAYS                                             [← Back]    │
│                                                                  │
│  Search... [⌕]  [My Games ▾] [Sort: Date ▾] [Filters ▾]        │
│                                                                  │
│  ┌─ LIST ──────────────────────────┬─ DETAIL ─────────────────┐ │
│  │                                 │                           │ │
│  │ ■ Coastal Fortress              │  MAP PREVIEW              │ │
│  │   You vs PlayerX · Victory      │  ┌─────────────┐         │ │
│  │   12:34 · IC Default · Ranked   │  │             │         │ │
│  │   +32 Elo · Jan 15              │  │  (minimap)  │         │ │
│  │                                 │  │             │         │ │
│  │ ■ Desert Arena FFA              │  └─────────────┘         │ │
│  │   4 players · 2nd place         │                           │ │
│  │   24:01 · Vanilla RA            │  PLAYERS                  │ │
│  │   Jan 14                        │  P1: You (Allied) — Win   │ │
│  │                                 │  P2: PlayerX (Soviet) — L │ │
│  │ ■ Imported: match.orarep        │                           │ │
│  │   OpenRA · Converted            │  Duration: 12:34          │ │
│  │                                 │  Balance: IC Default      │ │
│  │                                 │  Speed: Normal            │ │
│  │                                 │  Signed: Relay-certified  │ │
│  │                                 │  Mod: (none)              │ │
│  │                                 │                           │ │
│  │                                 │  [Watch]  [Share]         │ │
│  │                                 │  [Rename] [Delete]        │ │
│  └─────────────────────────────────┴───────────────────────────┘ │
│                                                                  │
│  [Import Replay...]  [Enter Match ID...]  [Reset Filters]        │
└──────────────────────────────────────────────────────────────────┘
```

#### Filter System

Seven filter dimensions (adapted from OpenRA's proven model, extended with IC-specific fields):

| Filter         | Options                                                                        | Notes                                  |
| -------------- | ------------------------------------------------------------------------------ | -------------------------------------- |
| **Scope**      | My Games / Bookmarked / All Local / Imported                                   | Default: My Games                      |
| **Game Type**  | Any / Ranked / Custom / Campaign / Skirmish vs AI                              |                                        |
| **Date Range** | Today / This Week / This Month / This Year / All Time                          |                                        |
| **Duration**   | Any / Short (<10 min) / Medium (10–30 min) / Long (30–60 min) / Epic (60+ min) |                                        |
| **Map**        | Dropdown populated from local replay metadata                                  | Searchable                             |
| **Player**     | Text field with autocomplete from local replay metadata                        |                                        |
| **Outcome**    | Any / Victory / Defeat / Draw                                                  | Relative to the selected player filter |

- Sort by: Date (default), Duration, Map Name, Player Count, Rating Change
- Filters are additive (AND logic); [Reset Filters] clears all
- Replay list loads asynchronously — no UI freeze on large collections

#### Replay Detail Panel (Right Side)

- **Map preview:** Minimap render with spawn point markers per player (colored dots)
- **Player list:** Name, faction, team, outcome (Win/Loss/Draw), APM average
- **Metadata:** Duration, balance preset, game speed, mod fingerprint, signed/unsigned status, engine version
- **Missing map handling:** If the replay's map is not installed, show [Install Map →] inline (downloads from Workshop if available) — adapted from OpenRA's auto-install pattern
- **Foreign replay badge:** Imported replays show source format badge (OpenRA / Remastered) and divergence confidence level (D056)

#### Actions

| Button                  | Action                                                                 |
| ----------------------- | ---------------------------------------------------------------------- |
| **[Watch]**             | Launch Replay Viewer                                                   |
| **[Share]**             | Copy Match ID to clipboard, or export `.icrep` file                    |
| **[Rename]**            | Rename the replay file                                                 |
| **[Delete]**            | Delete with confirmation                                               |
| **[Import Replay...]**  | File browser for `.icrep`, `.orarep`, Remastered replays (D056)        |
| **[Enter Match ID...]** | Download a relay-hosted replay by match ID (see Sharing section below) |

---

### Replay Viewer

```
Replay Browser → [Watch]
  — or —
Post-Game → [Watch Replay]
  — or —
Match History → [Watch]
  — or —
Double-click .icrep file (OS file association)
```

The Replay Viewer reuses the full game viewport with an observer transport bar replacing the player HUD. The right sidebar shows the minimap and observer panels.

#### Layout

```
┌────────────────────────────────────────────────┬──────────────────┐
│                                                │    MINIMAP        │
│                                                │   (clickable)     │
│              GAME VIEWPORT                     │                  │
│           (replay playback)                    ├──────────────────┤
│                                                │  OBSERVER PANEL   │
│                                                │  (toggleable,     │
│                                                │   see § Overlays) │
│                                                │                  │
│                                                │                  │
│                                                │                  │
├────────────────────────────────────────────────┴──────────────────┤
│  TRANSPORT BAR                                                    │
│                                                                   │
│  ⏮ ◄◄ ◄ ▶/⏸ ► ►► ⏭    0.5x [1x] 2x 4x 8x    12:34 / --:--    │
│                                                                   │
│  ├─△──●──△────△─────△──────────────────────────────────────┤     │
│    ⚔     ⚔🏠  ⚔⚔   🏠                                           │
│                                                                   │
│  CAMERA: [P1 ▾] [P2] [Free] [Follow Unit] [Directed]  [Fog ▾]   │
│  PANELS: [A]rmy [P]rod [E]con [Po]wers [S]core [AP]M  [Voice ▾] │
│                                                                   │
│  [Bookmark] [Clip] [Summary]                          [Settings]  │
└───────────────────────────────────────────────────────────────────┘
```

---

### Transport Controls

#### Button Bar

| Button               | Icon | Action                                                  |
| -------------------- | ---- | ------------------------------------------------------- |
| **Jump to Start**    | ⏮    | Jump to tick 0                                          |
| **Rewind 15s**       | ◄◄   | Jump back 15 seconds (configurable: 5s/10s/15s/30s)     |
| **Step Back**        | ◄    | Step back one game tick (hold for slow reverse scan)    |
| **Play / Pause**     | ▶/⏸  | Toggle playback                                         |
| **Step Forward**     | ►    | Step forward one game tick (hold for slow forward scan) |
| **Fast Forward 15s** | ►►   | Jump forward 15 seconds                                 |
| **Jump to End**      | ⏭    | Jump to final tick                                      |

#### Speed Controls

Speed buttons are displayed as discrete clickable labels (not a dropdown — LoL's single-click model, avoiding Dota 2's dropdown regression):

```
0.25x  0.5x  [1x]  2x  4x  8x  Max
```

The active speed is highlighted. Click to switch instantly.

#### Keyboard Shortcuts

| Key                 | Action                                           |
| ------------------- | ------------------------------------------------ |
| `Space`             | Play / Pause                                     |
| `B`                 | Rewind 15 seconds (SC2 convention)               |
| `N`                 | Fast forward 15 seconds                          |
| `,` (comma)         | Step back one tick                               |
| `.` (period)        | Step forward one tick                            |
| `[`                 | Decrease speed one tier                          |
| `]`                 | Increase speed one tier                          |
| `Home`              | Jump to start                                    |
| `End`               | Jump to end                                      |
| `Ctrl+B`            | Add bookmark at current tick                     |
| `←` / `→`           | Jump to previous / next bookmark or event marker |
| `Ctrl+←` / `Ctrl+→` | Jump to previous / next engagement               |
| `Escape`            | Open replay menu (exit / settings / summary)     |

#### Seeking

IC's 300-tick (~10 second) keyframe snapshots enable **true arbitrary seeking** — the key architectural advantage over OpenRA, AoE2, CoH3, and WC3 (all forward-only).

- **Click anywhere on the timeline** to jump to that moment
- **Drag the playhead** to scrub through the replay
- Re-simulation from nearest keyframe takes <100ms for typical games
- Both forward and backward seeking are supported
- Seeking works at any point — no "already viewed" restriction (unlike SC2)

---

### Timeline / Scrub Bar

The timeline is the most important UX element in the replay viewer. IC's design draws from LoL's annotated timeline (the strongest surveyed) while adding spoiler-free mode (an unmet need across all games).

#### Layout

```
├─△──●──△────△─────△──────────────────────────────────────┤
  ⚔     ⚔🏠  ⚔⚔   🏠
```

- **Horizontal progress bar** spanning the full width of the transport area
- **Playhead** (●) shows current position; draggable
- **Event markers** (△) appear as small icons above the bar at their timestamp
- **Engagement zones** shown as colored intensity bands behind the bar (SC2 combat shading pattern)
- **Time display:** `elapsed / total` in `mm:ss` format (or `--:--` in spoiler-free mode)

#### Event Markers

Auto-generated from the analysis event stream (see `formats/save-replay-formats.md` § "Analysis Event Stream"):

| Marker                           | Icon         | Source Event                                                                        |
| -------------------------------- | ------------ | ----------------------------------------------------------------------------------- |
| **Unit destroyed (significant)** | ⚔            | `UnitDestroyed` (filtered to non-trivial: hero units, expensive units, first blood) |
| **Base structure destroyed**     | 🏠            | `ConstructionDestroyed` for production/defense buildings                            |
| **Tech transition**              | ▲            | `UpgradeCompleted` for tier-changing upgrades                                       |
| **Expansion established**        | ◆            | `ConstructionCompleted` for resource structures at new locations                    |
| **Engagement zone**              | Colored band | Clusters of `UnitDestroyed` events within a time window                             |
| **Player eliminated**            | ☓            | `PlayerEliminated` / `MatchEnded` for that player                                   |
| **Bookmark (user)**              | 🔖            | User-placed via `Ctrl+B`                                                            |

#### Contextual Highlighting (LoL Pattern)

When the viewer locks camera to a specific player:

- That player's event markers **brighten** on the timeline
- Other players' markers **fade** (not hidden — just reduced opacity)
- This makes "find my kills" or "find my losses" effortless without separate filtering UI

#### Spoiler-Free Mode

**No game surveyed offers this.** IC can be first.

When enabled (toggle in replay menu or Settings → Gameplay):

- Total duration display shows `--:--` instead of the actual end time
- Timeline bar renders as an **expanding bar** that only shows the elapsed portion (the unplayed portion is hidden, not greyed out)
- Event markers only appear for already-viewed portions
- The progress bar does not reveal how much game remains

```
Spoiler-Free ON:
├─△──●──△────△─                                           │
  ⚔     ⚔🏠  ⚔⚔   ← bar ends at playhead; future is hidden
                    12:34 / --:--

Spoiler-Free OFF (default):
├─△──●──△────△─────△──────────────────────────────────────┤
  ⚔     ⚔🏠  ⚔⚔   🏠                    12:34 / 38:21
```

**Default:** Spoiler-free is **off** by default (most users want the full timeline). The setting persists across sessions.

---

### Camera Modes

Six camera modes, selectable via the camera bar or hotkeys:

| Mode                   | Key                       | Description                                                                                                                                                                                                                     |
| ---------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Free Camera**        | `F`                       | Default. Pan (edge scroll / middle-click drag / WASD), zoom (scroll wheel), minimap click to jump. Standard RTS observer camera.                                                                                                |
| **Player Perspective** | `1`–`8`                   | Lock to a specific player's recorded camera position, scroll, and zoom. Shows what the player actually saw during the game. Selected units shown with dashed circles in player color (SC2 pattern).                             |
| **Follow Unit**        | `Ctrl+F` on selected unit | Camera tracks a selected unit, keeping it centered. Useful for following hero units, harvesters, scouts. Click elsewhere or press `F` to exit.                                                                                  |
| **Directed Camera**    | `D`                       | AI-controlled camera that automatically follows the action. Jumps between engagements, expansions, and production events. Useful for passive viewing and casting. Configurable aggression (how quickly it cuts between events). |
| **Drone Follow**       | `Ctrl+D`                  | Loosely attached camera that follows the action with inertia and smooth movement. Cinematic feel without sharp cuts. (Fortnite drone-attach pattern adapted for isometric RTS.)                                                 |
| **All Vision**         | `0`                       | Free camera with fog/shroud disabled for all players. Shows the full map state.                                                                                                                                                 |

#### Vision / Fog-of-War Controls

Dropdown in the camera bar:

| Option                     | Key       | Effect                                            |
| -------------------------- | --------- | ------------------------------------------------- |
| **All Players (Combined)** | `-`       | See the union of all players' vision              |
| **Disable Shroud**         | `=`       | Full map revealed, including cloaked/hidden units |
| **Player 1 Vision**        | `Shift+1` | See only what Player 1 can see                    |
| **Player 2 Vision**        | `Shift+2` | See only what Player 2 can see                    |
| ...                        | `Shift+N` | Per-player fog-of-war                             |

**Ghost View (analysis mode):** `Ctrl+=` — Full map revealed, but units outside the selected player's vision are shown as translucent ghosts. Useful for studying opponent movements you couldn't see during the game. (Adapted from CS2 X-ray concept for RTS.)

---

### Observer Overlay Panels

Hotkey-toggled panels in the right sidebar. Each panel is independently toggleable and shows real-time data for all players. Design follows SC2's proven model — the most praised RTS observer system across all games surveyed.

| Panel          | Key | Content                                                                                                                                        |
| -------------- | --- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Army**       | `A` | Per-player army composition: unit type icons with counts and total army value (resource cost of living military units). Color-coded by player. |
| **Production** | `P` | Per-player active build queues: what each player is currently building (units, structures, upgrades) with progress bars.                       |
| **Economy**    | `E` | Per-player resource counts: credits on hand, income rate ($/min), harvester count, refinery count.                                             |
| **Powers**     | `W` | Per-player support power status: available/recharging/locked. Timer bars for recharging powers.                                                |
| **Score**      | `S` | Per-player score breakdown: Units Destroyed value, Units Lost value, Structures Destroyed, Structures Lost.                                    |
| **APM**        | `M` | Per-player Actions Per Minute: current window APM and game-average APM. Bar graph or sparkline.                                                |

#### Panel Display Modes

Three display density options (CoH3 pattern):

| Mode         | Description                                         | Use Case                                   |
| ------------ | --------------------------------------------------- | ------------------------------------------ |
| **Expanded** | Full panel detail with all data visible             | Learning, analysis                         |
| **Compact**  | Condensed single-line-per-player summary            | Experienced viewers wanting viewport space |
| **Caster**   | Side-by-side team comparison layout, minimal chrome | Broadcast / streaming                      |

Toggle with `Tab` to cycle modes, or select from replay settings.

#### Larger Broadcast Panels (SC2 `Ctrl+` Pattern)

For tournament broadcasts and streaming, larger center-screen panels:

| Key      | Panel                                           |
| -------- | ----------------------------------------------- |
| `Ctrl+A` | Large Army + Worker supply comparison           |
| `Ctrl+E` | Large Income comparison                         |
| `Ctrl+S` | Large Score comparison                          |
| `Ctrl+N` | Player Name banners (name, faction, team color) |

These overlay the center-top of the viewport and auto-hide after 5 seconds (or on any key press). Designed for broadcast transitions.

---

### Graphs and Analysis Overlays

Available via the [Summary] button or during playback as overlay panels:

#### Timeline Graphs (SC2 Game Summary Pattern)

| Graph                 | Content                                           | Notes                                                                  |
| --------------------- | ------------------------------------------------- | ---------------------------------------------------------------------- |
| **Army Value**        | Total military resource cost per player over time | Engagement zones shown as colored bands where army value drops sharply |
| **Income**            | Per-minute harvesting rate per player over time   | Shows economic advantage shifts                                        |
| **Unspent Resources** | Credits on hand per player over time              | High unspent = floating resources (coaching signal)                    |
| **Workers**           | Harvester count per player over time              | Economic investment tracking                                           |
| **APM**               | Actions Per Minute per player over time           | Activity patterns and fatigue                                          |

Graphs are clickable — click a point on any graph to jump the replay to that timestamp.

#### Build Order Timeline

Side-by-side per-player build order timeline showing:

- Unit and structure production events as icons on a horizontal time axis
- Upgrade completions marked with arrows
- Gap periods visible (idle production = coaching signal)

#### Heatmaps (Analysis Mode)

Accessible via [Summary] → Heatmaps tab:

| Heatmap              | Content                                                                           |
| -------------------- | --------------------------------------------------------------------------------- |
| **Unit Death**       | Where units died on the map (red = high density)                                  |
| **Combat**           | Where engagements occurred (intensity = resource cost traded)                     |
| **Camera Attention** | Where the player's camera spent time (from `CameraPositionSample` events at 2 Hz) |
| **Economy**          | Where harvesters operated (resource field usage patterns)                         |

Heatmaps render as semi-transparent overlays on the minimap or full viewport.

---

### Replay Bookmarks

Users can mark moments for later reference:

- **Add bookmark:** `Ctrl+B` at current playhead position
- **Name bookmark:** Optional text label (default: `Bookmark at mm:ss`)
- **Navigate:** `←` / `→` to jump between bookmarks and event markers
- **Bookmark list:** Accessible via the replay menu; shows timestamp + label for each
- **Persistent:** Bookmarks are saved alongside the replay file (in a sidecar `.icrep.bookmarks` JSON file, not modifying the replay itself)

---

### Voice Playback

If voice was recorded during the match (opt-in per D059 consent model):

- **Per-player voice tracks** toggle in the transport bar: `[Voice ▾]` dropdown lists each player's name with a checkbox
- Voice tracks are Opus-encoded, aligned to game ticks
- **Mute/unmute** individual players without affecting others
- **Volume slider** per track (accessible from dropdown)
- Voice playback automatically syncs with replay speed (pitch-corrected at 2x; muted above 4x)

---

### Post-Game Summary Screen

Accessible from:
- Post-Game → [Summary] (after a live match)
- Replay Viewer → [Summary] button (during or after replay playback)
- Replay Browser → right-click → [View Summary]

The summary screen does **not** require replaying the match — it reads from the analysis event stream embedded in the `.icrep` file.

```
┌──────────────────────────────────────────────────────────────────┐
│  MATCH SUMMARY — Coastal Fortress                     [← Back]   │
│                                                                  │
│  P1: You (Allied) — VICTORY     P2: PlayerX (Soviet) — DEFEAT   │
│  Duration: 12:34   Balance: IC Default   Speed: Normal           │
│                                                                  │
│  [Overview] [Economy] [Military] [Build Order] [Heatmaps]        │
│  ─────────────────────────────────────────────────────────────── │
│                                                                  │
│  OVERVIEW                                                        │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Army Value Graph (over time)                              │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │     ╱\   P1                    ╱\                   │  │ │
│  │  │    ╱  \  ──── P2          ╱\  ╱  \                  │  │ │
│  │  │   ╱    ╲╱   ╲           ╱  ╲╱    ╲                 │  │ │
│  │  │  ╱           ╲     ╱───╱         ╲                 │  │ │
│  │  │ ╱             ╲───╱               ╲___             │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  │  (click graph to jump to that moment in replay)           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  SCORE SUMMARY                                                   │
│  ┌──────────────┬──────────┬──────────┐                         │
│  │              │ You      │ PlayerX  │                         │
│  ├──────────────┼──────────┼──────────┤                         │
│  │ Units Killed │ 47       │ 23       │                         │
│  │ Units Lost   │ 31       │ 52       │                         │
│  │ Structures   │ 3 / 1    │ 1 / 5    │  (destroyed / lost)    │
│  │ Income Total │ $14,200  │ $11,800  │                         │
│  │ APM (avg)    │ 86       │ 62       │                         │
│  └──────────────┴──────────┴──────────┘                         │
│                                                                  │
│  [Watch Replay]  [Share]  [Export Summary]                        │
└──────────────────────────────────────────────────────────────────┘
```

**Tabs:**
- **Overview:** Army value graph, score summary, key moments timeline
- **Economy:** Income graph, unspent resources graph, harvester count graph, total earned/spent
- **Military:** Per-unit-type kill/death/efficiency table, army composition pie charts at key moments
- **Build Order:** Side-by-side production timelines per player (adapted from SC2 Game Summary)
- **Heatmaps:** Unit death, combat, camera attention, economy heatmaps on minimap view

All graphs are clickable — click a point to open the Replay Viewer at that timestamp.

---

### Replay Sharing

#### Match ID System (Dota 2 Pattern)

Relay-hosted matches generate a unique **Match ID** (short alphanumeric hash, e.g., `IC-7K3M9X`). Any player can enter this ID in the replay browser to download the replay.

- **Copy Match ID:** Available in post-game screen, replay browser detail panel, and profile match history
- **[Enter Match ID...]** in replay browser: text field → download from relay → add to local library
- **URL format:** `ic://replay/IC-7K3M9X` — opens IC directly to the replay (OS URL scheme handler)
- **Availability:** Relay-hosted replays persist for a configurable period (default: 90 days, server-operator configurable via D072). After expiry, only locally-saved copies remain.
- **Privacy:** Match IDs for ranked games are public by default. Custom/private games generate IDs only if the host enables sharing.

#### File-Based Sharing

- `.icrep` files are portable and self-describing
- **Embedded resources mode** (see `formats/save-replay-formats.md` § "Embedded Resources"): Self-contained replays that include the map and rule snapshots, so the recipient does not need matching content installed
- **File association:** `.icrep` registered with the OS; double-click opens IC's replay viewer
- **Drag-and-drop:** Drop an `.icrep` file onto the IC window to open it

#### Workshop Integration

- Community replays can be published to the Workshop as curated collections (e.g., "Best Games of Season 3", "Teaching Replays: Soviet Openings")
- Workshop replay packs include metadata for browsing without downloading every replay file
- Creators can attach commentary notes to published replays

---

### Video / Clip Export

**IC ships with built-in `.webm` video export** — ahead of every RTS surveyed except LoL's basic clip system.

#### Quick Clip

During replay playback:

1. Press `Ctrl+Shift+R` or click [Clip] to start recording
2. The transport bar shows a red recording indicator and elapsed clip time
3. Press `Ctrl+Shift+R` again to stop
4. Clip saved to `Replays/Clips/` as `.webm` (VP9 video + Opus audio)
5. Toast notification: `Clip saved (12s) — [Open Folder] [Copy to Clipboard]`

#### Full Replay Export

From replay browser or viewer menu: [Export Video...]

```
┌─────────────────────────────────────────────────────────────────┐
│  EXPORT REPLAY VIDEO                                             │
│                                                                 │
│  Range: [Full Replay ▾]  or  Start: [00:00] End: [12:34]        │
│                                                                 │
│  Resolution: [1920×1080 ▾]   Framerate: [60 fps ▾]              │
│  Quality:    [High ▾]        Format: [.webm (VP9) ▾]            │
│                                                                 │
│  Camera:  [Current camera settings ▾]                            │
│           (Free Camera / Player 1 / Player 2 / Directed)         │
│                                                                 │
│  Include:  ☑ Observer overlays   ☑ Transport bar (off for clean) │
│            ☑ Voice audio         ☑ Game audio                    │
│                                                                 │
│  Estimated size: ~180 MB   Estimated time: ~3 min                │
│                                                                 │
│  [Export]  [Cancel]                                               │
└─────────────────────────────────────────────────────────────────┘
```

**Render pipeline:** The export runs the replay at accelerated speed off-screen, capturing frames to the encoder. This allows higher-quality output than screen capture and works headless.

---

### Cinematic Camera Tools

For content creators and community filmmakers. Accessible via replay viewer menu → [Cinematic Mode] or `Ctrl+Shift+C`.

#### Camera Path Editor

Define a camera path with keyframes:

- Place keyframes at positions along the timeline (`Ctrl+K` to add keyframe at current camera position and tick)
- Each keyframe stores: camera position, zoom, rotation (for 3D render mode), playback speed at that point
- Camera interpolates smoothly between keyframes (Catmull-Rom spline)
- Preview the path before recording
- Export the camera path as a reusable `.iccam` file

#### Lens Controls (3D Render Mode, D048)

When using 3D render mode:

| Control          | Effect                                          |
| ---------------- | ----------------------------------------------- |
| **Focal Length** | Wide-angle to telephoto (adjustable slider)     |
| **Aperture**     | Depth-of-field blur amount (lower = more bokeh) |
| **Auto Focus**   | Toggle; when off, manual focus distance slider  |

#### Cinematic Toggles

| Toggle                | Key            | Effect                                                                |
| --------------------- | -------------- | --------------------------------------------------------------------- |
| **Hide all UI**       | `Ctrl+Shift+H` | Remove all overlays, transport bar, panels — clean game viewport only |
| **Hide player names** |                | Remove floating player/unit names                                     |
| **Hide health bars**  |                | Remove health/selection indicators                                    |
| **Letterbox**         |                | Add cinematic black bars (21:9 crop on 16:9 display)                  |

---

### Moddable Observer UI

The observer overlay system is **data-driven and moddable** (SC2 custom observer UI pattern). Community creators can publish custom observer layouts via the Workshop.

- Observer panel layouts are defined in YAML (position, size, data bindings, conditional visibility)
- The game provides a standardized data API that observer panels read from (player stats, army composition, economy, production, APM)
- Built-in layouts: `Default`, `Compact`, `Caster Broadcast`
- Workshop layouts installable and selectable from replay viewer settings
- Layout switching is instant (no reload required)

This enables community-created broadcast overlays (equivalent to SC2's WCS Observer and AhliObs) without engine modifications.

---

### Live Spectator Mode

Live spectating shares the same viewer infrastructure as replay playback, with these differences:

| Feature            | Replay Viewer                 | Live Spectator                                                                |
| ------------------ | ----------------------------- | ----------------------------------------------------------------------------- |
| Transport controls | Full (seek, rewind, speed)    | Play only; no rewind/seek (live stream)                                       |
| Speed              | 0.25x–8x + Max                | Real-time only                                                                |
| Broadcast delay    | N/A                           | Configurable (default 120s for ranked/tournament)                             |
| Observer panels    | All available                 | All available                                                                 |
| Camera modes       | All six                       | All six                                                                       |
| Voice              | Recorded tracks               | Live voice (if spectator permitted)                                           |
| Join timing        | Any time (file is complete)   | Must join before match or during (mid-game join supported via relay snapshot) |
| Chat               | N/A (replay has no live chat) | Observer chat channel (separate from player chat — anti-coaching per D059)    |

#### Mid-Game Spectator Join

Unlike OpenRA (which cannot do this), IC's relay architecture supports spectators joining a match in progress:

1. Spectator requests join via relay
2. Relay sends current state snapshot + recent order backlog
3. Client re-simulates from snapshot to catch up
4. Spectator enters live stream with <5 second catch-up delay

#### Spectator Slots

- Visible in lobby with spectator count / max slots
- Separate from player slots
- Lobby host configures: max spectators, fog-of-war policy, broadcast delay
- Tournament mode: spectator slots may require organizer approval

---

### Foreign Replay Playback (D056)

Imported replays (OpenRA `.orarep`, Remastered Collection) play through the same viewer with additional UX:

#### Divergence Confidence Indicator

A small badge in the transport bar shows the current divergence confidence level:

| Level           | Badge    | Meaning                                                                         |
| --------------- | -------- | ------------------------------------------------------------------------------- |
| **Plausible**   | Green ✓  | Replay is tracking well; no detectable divergence                               |
| **Minor Drift** | Yellow ⚠ | Small state differences detected; visuals may differ slightly from the original |
| **Diverged**    | Red ✗    | Significant divergence; replay may not accurately represent the original match  |

The badge is clickable to show a detail panel with divergence metrics and explanation.

#### Limitations Banner

Foreign replays show a subtle top banner on first load:

```
This replay was imported from {OpenRA / Remastered}. Playback uses translated
orders and may differ from the original. [Learn More] [Dismiss]
```

---

### Replay Anonymization

`ic replay anonymize <file>` (CLI) or Replay Browser → right-click → [Anonymize...]:

- Replace player names with generic labels (`Player 1`, `Player 2`, etc.)
- Strip voice tracks
- Strip chat messages
- Preserve all gameplay data (orders, events, state hashes)
- Useful for educational content sharing, tournament review, and privacy

---

### Replay Settings

Accessible via [Settings] gear icon in the transport bar:

| Setting                    | Options                             | Default          |
| -------------------------- | ----------------------------------- | ---------------- |
| **Spoiler-free mode**      | On / Off                            | Off              |
| **Rewind jump duration**   | 5s / 10s / 15s / 30s                | 15s              |
| **Auto-record all games**  | On / Off                            | On               |
| **Default camera mode**    | Free / Directed / Player 1          | Free             |
| **Default observer panel** | None / Army / Economy / Score       | None             |
| **Panel display density**  | Expanded / Compact / Caster         | Expanded         |
| **Event marker density**   | All / Significant Only / Off        | Significant Only |
| **Voice playback default** | All On / All Off / Per-Player       | All On           |
| **Observer UI layout**     | Default / Compact / Caster / Custom | Default          |

---

### Platform Adaptations

| Platform                 | Adaptation                                                                                                                  |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| **Desktop (KBM)**        | Full hotkey set; all features accessible                                                                                    |
| **Gamepad / Steam Deck** | Transport controls on D-pad; camera on sticks; panels on shoulder buttons; radial menu for camera modes                     |
| **Touch (Tablet)**       | Swipe timeline to scrub; pinch to zoom; tap event markers to jump; floating transport buttons; panels in collapsible drawer |
| **Phone**                | Simplified overlay with one panel at a time; timeline at bottom with large touch targets; speed control via tap zones       |

---

### Cross-References

| Topic                              | Document                                                          |
| ---------------------------------- | ----------------------------------------------------------------- |
| Replay file format (`.icrep`)      | `formats/save-replay-formats.md` § Replay File Format             |
| State recording and keyframes      | `architecture/state-recording.md`                                 |
| Analysis event stream              | `formats/save-replay-formats.md` § Analysis Event Stream          |
| Foreign replay import (D056)       | `decisions/09f/D056-replay-import.md`                             |
| Voice recording consent (D059)     | `decisions/09g/D059-communication.md`                             |
| Replay signatures and trust (D052) | `decisions/09b/D052-community-servers.md`                         |
| Observer/spectator mode (live)     | `player-flow/in-game.md` § Observer Overlays                      |
| Post-game flow                     | `player-flow/post-game.md`                                        |
| Netcode and replay architecture    | `03-NETCODE.md`                                                   |
| Cross-game replay UX survey        | `research/replay-playback-ux-survey.md`                           |
| LLM replay overlays (D073)         | `decisions/09d/D073-llm-exhibition-modes.md` § Spectator Overlays |
| Moddable UI system                 | `02-ARCHITECTURE.md` § UI Theme System                            |
