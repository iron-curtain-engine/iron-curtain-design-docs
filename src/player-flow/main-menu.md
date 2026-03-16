## Main Menu

The main menu is the hub. Everything is reachable from here. The shellmap plays behind a semi-transparent overlay panel.

### Layout

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│                    [ IRON CURTAIN ]                               │
│                    Red Alert                                     │
│                                                                  │
│              ┌─────────────────────────┐                         │
│              │  ► Continue Campaign     │ (if save exists)       │
│              │  ► Campaign              │                         │
│              │  ► Skirmish              │                         │
│              │  ► Multiplayer           │                         │
│              │                          │                         │
│              │  ► Replays               │                         │
│              │  ► Workshop              │                         │
│              │  ► Settings              │                         │
│              │                          │                         │
│              │  ► Profile               │ (bottom group)         │
│              │  ► Encyclopedia          │                         │
│              │  ► Credits               │                         │
│              │  ► Quit                  │                         │
│              └─────────────────────────┘                         │
│                                                                  │
│  [shellmap: live AI battle playing in background]                │
│                                                                  │
│  Iron Curtain v0.1.0        community.ironcurtain.dev    RA 1.0 │
└──────────────────────────────────────────────────────────────────┘
```

### Button Descriptions

| Button                | Action                                                            | Notes                                                                                                       |
| --------------------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Continue Campaign** | Resumes the campaign at its current authored progression point | If there is only one immediate next mission, one click launches it. If multiple missions are currently available or an urgent pending branch exists, it reopens the campaign map / intermission selection instead of choosing a node for the player. |
| **Campaign**          | Opens Campaign Selection screen                                   | Choose faction (Allied/Soviet), start new campaign, or select saved campaign slot.                          |
| **Skirmish**          | Opens Skirmish Setup screen                                       | Configure a local game vs AI: map, players, settings.                                                       |
| **Multiplayer**       | Opens Multiplayer Hub                                             | Five ways to find a game: Browser, Join Code, Ranked, Direct IP, QR Code.                                   |
| **Replays**           | Opens Replay Browser                                              | Browse saved replays, import foreign replays (.orarep, Remastered).                                         |
| **Workshop**          | Opens Workshop Browser                                            | Browse, install, manage mods/maps/resources from Workshop sources.                                          |
| **Settings**          | Opens Settings screen                                             | All configuration: video, audio, controls, experience profile, data, LLM.                                   |
| **Profile**           | Opens Player Profile                                              | View/edit identity, achievements, stats, friends, community memberships.                                    |
| **Encyclopedia**      | Opens in-game Encyclopedia                                        | Auto-generated unit/building reference from YAML rules.                                                     |
| **Credits**           | Shows credits sequence                                            | Scrolling credits, skippable.                                                                               |
| **Quit**              | Exits to desktop                                                  | Immediate — no "are you sure?" dialog (following the principle that the game respects the player's intent). |

### Contextual Elements

- **Version info** — Bottom-left: engine version, game module version
- **Community link** — Bottom-center: link to community site/Discord
- **Mod indicator** — If a non-default mod profile is active, a small indicator badge shows which profile (e.g., "Combined Arms v2.1")
- **News ticker** (optional, Modern theme) — Community announcements from the configured tracking server(s)
- **Tutorial hint** — For new players: a non-intrusive callout near Campaign or Skirmish saying "New? Try the tutorial → Commander School" (D065, dismissible, appears once)
- **Background selection** — Configurable via Settings → Video. Options: static theme image, shellmap AI battle (default for Remastered/Modern themes), personal highlights cycling from the player's highlight library (D077), community/tournament highlight packs from Workshop (D077), or **campaign-progress scene** (the background reflects the player's current campaign stage — see below). Falls back to shellmap AI if the selected option is unavailable. Highlight/campaign playback re-simulates from nearest keyframe at reduced priority behind menu UI

#### Campaign-Progress Menu Background

When the player has an active campaign, the main menu background can reflect where they are in the story — changing as they progress through missions. This is an [Evolving Title Screen](https://tvtropes.org/pmwiki/pmwiki.php/Main/EvolvingTitleScreen) pattern used by Half-Life 2, Halo: Reach, Spec Ops: The Line, Portal 2, The Last of Us Part II, Warcraft III, Lies of P, and others.

**How it works:** Campaign authors define a `menu_scenes` table in their campaign YAML (see `modding/campaigns.md` § Campaign Menu Scenes). Each entry maps a campaign progress point (mission ID, flag state, or completion percentage) to a menu scene. The scene can be:

- **A shellmap scenario** — a live, lightweight in-game scene (a few AI units fighting, a base under construction, an air patrol) rendered behind the menu. Uses the existing shellmap infrastructure with campaign-specific map/units/scripts
- **A video loop** — a pre-rendered or recorded `.webm` video playing in a loop (aircraft flying in formation at night, a war room briefing, a battlefield aftermath). Audio plays at reduced volume behind menu music
- **A static image** — campaign-specific artwork or screenshot for the current act/chapter

**Scene selection priority:**
1. If the player has manually configured a different background style (static image, shellmap AI, highlights), that takes precedence — campaign scenes are opt-in, not forced
2. If "Campaign Scene" is selected (or is the campaign's default), the engine matches the player's current `CampaignState` against the `menu_scenes` table and picks the matching scene
3. If no campaign is active or no scene matches, falls back to the theme's default (shellmap AI or static image)

**Prior art:**

| Game | How It Works | What IC Can Learn |
|------|-------------|-------------------|
| **Half-Life 2** | Menu shows an area from the most recent chapter. Each chapter has a different background scene | Direct model — IC maps campaign progress to scenes |
| **Halo: Reach** | Menu artwork changes based on which campaign mission was last played. Uses concept art pieces | Supports both live scenes AND static art per mission |
| **Spec Ops: The Line** | Menu tableau evolves across the story — soldier sleeping → recon → combat → fire → destruction. Day turns to night. The menu IS a scene happening alongside the story | The menu scene can tell its own micro-story that parallels the campaign |
| **Warcraft III** | Each campaign (Human, Undead, Orc, Night Elf) has its own menu background and music | Per-campaign theming, not just per-mission |
| **Portal 2** | Menu shows a location from the current chapter — acts as a bookmark | Reinforces where the player left off |
| **The Last of Us Part II** | Menu evolves from calm boat scene → locked-down darkness → bright sunrise after completion | Emotional arc in the menu itself |
| **Lies of P** | Title screen shifts through different locations as the player reaches new chapters | Location-based scene changes |
| **Call of Duty** (classic) | Menu weapons change per campaign faction (Thompson for US, Lee-Enfield for UK, Mosin-Nagant for USSR) | Even small thematic details (faction-specific props) add immersion |
