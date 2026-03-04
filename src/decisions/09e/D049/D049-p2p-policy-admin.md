| ------------ | --------------- | ------------------------------------------------------------- |
| < 5MB        | N/A — HTTP only | P2P overhead exceeds benefit                                  |
| 5–50MB       | 256KB           | Fine-grained. Good for partial recovery and slow connections. |
| 50–500MB     | 1MB             | Balanced. Reasonable metadata overhead.                       |
| > 500MB      | 4MB             | Reduced metadata overhead for large packages.                 |

*Bandwidth limiting:* Configurable per-client in `settings.toml`. Residential users cannot have their connection saturated by mod seeding — this is a hard requirement that Kraken solves with `egress_bits_per_sec`/`ingress_bits_per_sec` and IC must match.

```toml
# settings.toml — P2P bandwidth configuration
[workshop.p2p]
max_upload_speed = "1 MB/s"          # Default. 0 = unlimited, "0 B/s" = no seeding
max_download_speed = "unlimited"      # Default. Most users won't limit.
seed_after_download = true            # Keep seeding while game is running
seed_duration_after_exit = "30m"      # Background seeding after game closes (0 = none)
cache_size_limit = "2 GB"             # LRU eviction when exceeded
prefer_p2p = true                     # false = always use HTTP direct
```

*Health checks:* Seed boxes implement heartbeat health checks (30s interval, 3 failures → unhealthy, 2 passes → healthy again — matching Kraken's active health check parameters). The tracker marks peers as offline after 2× announce interval without contact. Unhealthy seed boxes are removed from the announce response until they recover.

*Content lifecycle:* Downloaded packages stay in the seeding pool for 30 minutes after the game exits (configurable via `seed_duration_after_exit`). This is longer than Kraken's 5-minute `seeder_tti` because IC has fewer peers per package — each seeder is more valuable. Disk cache uses LRU eviction when over `cache_size_limit`. Packages currently in use or being seeded are never evicted.

*Download priority tiers:* Inspired by Dragonfly's 7-level priority system (Level0–Level6), IC uses 3 priority tiers to enable QoS differentiation. Higher-priority downloads preempt lower-priority ones (pause background downloads, reallocate bandwidth and connection slots):

| Priority | Name             | When Used                                                | Behavior                                                   |
| -------- | ---------------- | -------------------------------------------------------- | ---------------------------------------------------------- |
| 1 (high) | `lobby-urgent`   | Player joining a lobby that requires missing mods        | Preempts all other downloads. Uses all available bandwidth |
| 2 (mid)  | `user-requested` | Player manually downloads from Workshop browser          | Normal bandwidth. Runs alongside background.               |
| 3 (low)  | `background`     | Cache warming, auto-updates, subscribed mod pre-download | Bandwidth-limited. Paused when higher-priority active.     |

*Preheat / prefetch:* Adapted from Dragonfly's preheat jobs (which pre-warm content on seed peers before demand). IC uses two prefetch patterns:

- **Lobby prefetch:** When a lobby host sets required mods, the Workshop server (Phase 5+) can pre-seed those mods to seed boxes before players join. The lobby creation event is the prefetch signal. This ensures seed infrastructure is warm when players start downloading.
- **Subscription prefetch:** Players can subscribe to Workshop publishers or resources. Subscribed content auto-downloads in the background at `background` priority. When a subscribed mod updates, the new version downloads automatically before the player next launches the game.

*Persistent replica count (Phase 5+):* Inspired by Dragonfly's `PersistentReplicaCount`, the Workshop server tracks how many seed boxes hold each resource. If the count drops below a configurable threshold (default: 2 for popular resources, 1 for all others), the server triggers automatic re-seeding from HTTP origin. This ensures the "always available" guarantee — even if all player peers are offline, seed infrastructure maintains minimum replica coverage.

**Early-phase bootstrap — Git-hosted package index:**

Before the full Workshop server is built (Phase 4-5), a **GitHub-hosted package index repository** serves as the Workshop's discovery and coordination layer. This is a well-proven pattern — Homebrew (`homebrew-core`), Rust (`crates.io-index`), Winget (`winget-pkgs`), and Nixpkgs all use a git repository as their canonical package index.

**How it works:**

A public GitHub repository (e.g., `iron-curtain/workshop-index`) contains YAML manifest files — one per package — that describe available resources, their versions, checksums, download locations, and dependencies. The repo itself contains NO asset files — only lightweight metadata.

```
workshop-index/                      # The git-hosted package index
├── index.yaml                       # Consolidated index (single-fetch for game client)
├── packages/
│   ├── alice/
│   │   └── soviet-march-music/
│   │       ├── 1.0.0.yaml           # Per-version manifests
│   │       └── 1.1.0.yaml
│   ├── community-hd-project/
│   │   └── allied-infantry-hd/
│   │       └── 2.0.0.yaml
│   └── ...
├── sources.yaml                     # List of storage servers, mirrors, seed boxes
└── .github/
    └── workflows/
        └── validate.yml             # CI: validates manifest format, checks SHA-256
```

**Per-package manifest (`packages/alice/soviet-march-music/1.1.0.yaml`):**

```yaml
name: soviet-march-music
publisher: alice
version: 1.1.0
license: CC-BY-4.0
description: "Soviet faction battle music pack"
size: 48_000_000  # 48MB
sha256: "a1b2c3d4..."

sources:
  - type: http
    url: "https://github.com/iron-curtain/workshop-packages/releases/download/alice-soviet-march-music-1.1.0/soviet-march-music-1.1.0.icpkg"
  - type: torrent
    info_hash: "e5f6a7b8..."
    trackers:
      - "wss://tracker.ironcurtain.gg/announce"   # WebTorrent tracker
      - "udp://tracker.ironcurtain.gg:6969/announce"

dependencies:
  community-hd-project/base-audio-lib: "^1.0"

game_modules: [ra]
tags: [music, soviet, battle]
```

**`sources.yaml` — storage server and tracker registry:**

```yaml
# Where to find actual .icpkg files and BitTorrent peers.
# The engine reads this to discover available download sources.
# Adding an official server later = adding a line here.
storage_servers:
  - url: "https://github.com/iron-curtain/workshop-packages/releases"  # GitHub Releases (Phase 0-3)
    type: github-releases
    priority: 1
  # - url: "https://cdn.ironcurtain.gg"   # Future: official CDN (Phase 5+)
  #   type: http
  #   priority: 1

torrent_trackers:
  - "wss://tracker.ironcurtain.gg/announce"      # WebTorrent (browser + desktop)
  - "udp://tracker.ironcurtain.gg:6969/announce"  # UDP (desktop only)

seed_boxes:
  - "https://seed1.ironcurtain.gg"  # Permanent seeder for all packages
```

**Two client access patterns:**

1. **HTTP fetch** (game client default): The engine fetches `index.yaml` via `raw.githubusercontent.com` — a single GET request returns the full package listing. Fast, no git dependency, CDN-backed globally by GitHub. Cached locally with ETag/Last-Modified for incremental updates.
2. **Git clone/pull** (SDK, power users, offline): `git clone` the entire index repo. `git pull` for incremental atomic updates. Full offline browsing. Better for the SDK/editor and users who want to script against the index.

The engine's Workshop source configuration (D030) treats this as a new source type:

```toml
# settings.toml — Phase 0-3 configuration
[[workshop.sources]]
url = "https://github.com/iron-curtain/workshop-index"   # git-index source
type = "git-index"
priority = 1

[[workshop.sources]]
path = "C:/my-local-workshop"    # local development
type = "local"
priority = 2
```

**Community contribution workflow (manual):**

1. Modder creates a `.icpkg` package and uploads it to GitHub Releases (or any HTTP host)
2. Modder submits a PR to `workshop-index` adding a manifest YAML with SHA-256 and download URL
3. GitHub Actions validates manifest format, checks SHA-256 against the download URL, verifies metadata
4. Maintainers review and merge → package is discoverable to all players on next index fetch
5. When the full Workshop server ships (Phase 4-5), published packages migrate automatically — the manifest format is the same

**Git-index security hardening** (see `06-SECURITY.md` § Vulnerabilities 20–21 and `research/workshop-registry-vulnerability-analysis.md` for full threat analysis):

- **Path-scoped PR validation:** CI rejects PRs that modify files outside the submitter's package directory. A PR adding `packages/alice/tanks/1.0.0.yaml` may ONLY modify files under `packages/alice/`. Modification of other paths → automatic CI failure.
- **CODEOWNERS:** Maps `packages/alice/** @alice-github`. GitHub enforces that only the package owner can approve changes to their manifests.
- **`manifest_hash` verification:** CI downloads the `.icpkg`, extracts `manifest.yaml`, computes its SHA-256, and verifies it matches the `manifest_hash` field in the index entry. Prevents manifest confusion (registry entry diverging from package contents).
- **Consolidated `index.yaml` is CI-generated:** Deterministically rebuilt from per-package manifests — never hand-edited. Any contributor can reproduce locally to verify integrity.
- **Index signing (Phase 3–4):** CI signs the consolidated `index.yaml` with an Ed25519 key stored outside GitHub. Clients verify the signature. Repository compromise without the signing key produces unsigned (rejected) indexes. Uses the **two-key architecture** from D052 (§ Key Lifecycle): the CI-held key is the Signing Key (SK); a Recovery Key (RK), held offline by ≥2 maintainers, enables key rotation on compromise without breaking client trust chains. See D052 § "Cross-Pollination" for the full rationale.
- **Actions pinned to commit SHAs:** All GitHub Actions referenced by SHA, not by mutable tag. Minimal `GITHUB_TOKEN` permissions. No secrets in the PR validation pipeline.
- **Branch protection on main:** Require signed commits, no force-push, require PR reviews, no single-person merge. Repository must have ≥3 maintainers.

**Automated publish via `ic` CLI (same UX as Phase 5+):**

The `ic mod publish` command works against the git-index backend in Phase 0–3:

1. `ic mod publish` packages content into `.icpkg`, computes SHA-256
2. Uploads `.icpkg` to GitHub Releases (via GitHub API, using a personal access token configured in `ic auth`)
3. Generates the index manifest YAML from `mod.yaml` metadata
4. Opens a PR to `workshop-index` with the manifest file
5. Modder reviews the PR and confirms; GitHub Actions validates; maintainers merge

The command is identical to Phase 5+ publishing (`ic mod publish`) — the only difference is the backend. When the Workshop server ships, `ic mod publish` targets the server instead. Modders don't change their workflow.

**Adding official storage servers later:**

When official infrastructure is ready (Phase 5+), adding it is a one-line change to `sources.yaml` — no architecture change, no client update. The `sources.yaml` in the index repo is the single place that lists where packages can be downloaded from. Community mirrors and CDN endpoints are added the same way.

**Phased progression:**

1. **Phase 0–3 — Git-hosted index + GitHub Releases:** The index repo is the Workshop. Players fetch `index.yaml` for discovery, download `.icpkg` files from GitHub Releases (2GB per file, free, CDN-backed). Community contributes via PR. Zero custom server code. Zero hosting cost.
2. **Phase 3–4 — Add BitTorrent tracker:** A minimal tracker binary goes live ($5-10/month VPS). Package manifests gain `torrent` source entries. P2P delivery begins for large packages. The index repo remains the discovery layer.
3. **Phase 4–5 — Full Workshop server:** Search, ratings, dependency resolution, FTS5, integrated P2P tracker. The Workshop server can either replace the git index or coexist alongside it (both are valid D030 sources). The git index remains available as a fallback and for community-hosted Workshop servers.

The progression is smooth because the federated source model (D030) already supports multiple source types — `git-index`, `local`, `remote` (Workshop server), and `steam` all coexist in `settings.toml`.

### Freeware / Legacy C&C Mirror Content (Policy-Gated, Not Assumed)

IC may choose to host **official/community mirror packages** for legacy/freeware C&C content, but this is a **policy-gated path**, not a default assumption.

Rules:
- Do **not** assume "freeware" automatically means "redistributable in IC Workshop mirrors."
- The default onboarding path remains **owned-install import** via D069 (including out-of-the-box Remastered import when detected).
- Mirroring legacy/freeware C&C assets in Workshop requires the D037 governance/legal policy gate:
  - documented rights basis / scope
  - provenance labeling
  - update/takedown process
  - mirror operator responsibilities
- If approved, mirrored packs must be clearly labeled (e.g., `official-mirror` / verified publisher/community mirror provenance) and remain optional content sources under D068/D069.

This preserves legal clarity without blocking player onboarding or selective-install workflows.

Rendered cutscene sequence bundles (D038 `Cinematic Sequence` content plus dialogue/portrait/audio/visual dependencies) are normal Workshop resources under the same D030/D049 rules. They should declare optional visual dependencies explicitly (for example HD/3D render-mode packs) and provide fallback-safe behavior so a scenario/campaign can still proceed when optional presentation packs are missing.

### Media Language Capability Metadata (Cutscenes / Voice / Subtitles / CC)

Workshop media packages that contain cutscenes, dubbed dialogue, subtitles, or closed captions should declare a **language capability matrix** so clients can make correct fallback decisions before playback.

Examples of package-level metadata (exact field names can evolve, semantics are fixed):

- `audio_languages[]` (dubbed/spoken audio languages available in this package)
- `subtitle_languages[]` (subtitle text languages available)
- `cc_languages[]` (closed-caption languages available)
- `translation_source` (`human`, `machine`, `hybrid`)
- `translation_quality_label` / trust label (e.g., `creator-verified`, `community-reviewed`, `machine-translated`)
- `coverage` (`full`, `partial`, or percentage by track/group)
- `requires_original_audio_pack` (for subtitle/CC-only translation packs)

Rules:

- Language capability metadata must be **accurate enough for fallback selection** (D068/D069) and player trust.
- Machine-translated subtitle/CC resources must be **clearly labeled** in Workshop listings, Installed Content Manager, and playback fallback notices.
- Missing language support must never block campaign progression; D068 fallback-safe behavior remains the rule.
- Media language metadata is **presentation scope** and does not affect gameplay compatibility fingerprints.

Workshop UX implications:

- Browse/search filters may include language availability badges (e.g., `Audio: EN`, `Subs: EN/HE`, `CC: EN/AR`).
- Package details should show translation source/trust labels and coverage.
- Install/enable flows should warn when a selected package does not satisfy the player's preferred cutscene voice/subtitle/CC preferences.

Operator/admin implications:

- The Workshop admin panel (M9) should surface language metadata and translation-source labels in package review/provenance screens so mislabeled machine translations or incomplete language claims can be corrected/quarantined.

### Workshop Operator / Admin Panel (Phased)

A full Workshop platform needs a dedicated **operator/admin panel** (web UI or equivalent admin surface), with CLI parity for automation.

#### Phase 4–5 / `M8` — Minimal Operator Panel (`P-Scale`)

Purpose: keep the Workshop running and recoverable before the full creator ecosystem matures.

Minimum operator capabilities:
- ingest/publish job queue status (pending / failed / retry)
- package hash verification status and retry actions
- source/index health (git-index sync, HTTP origins, tracker health)
- metadata/index rebuild and cache maintenance actions
- storage/CAS usage summary + GC triggers
- basic audit log of operator actions

#### Phase 6a / `M9` — Full Workshop Admin Panel (`P-Scale`)

Purpose: support moderation, provenance, release-channel controls, and ecosystem governance at scale.

Required admin capabilities:
- moderation queue (reports, quarantines, takedowns, reinstatements)
- provenance/license review queue and publish-readiness blockers
- signature/verification status dashboards (manifest/index/release metadata authenticity)
- dependency impact view ("what breaks if this package is quarantined/yanked?")
- release channel controls (`private` / `beta` / `release`)
- rollback/quarantine tools and incident notes
- role-based access control (operators/moderators/admins)
- append-only audit trail / action history

#### Phase 7 / `M11` — Governance & Policy Analytics

- moderation workload metrics and SLA views
- abuse/fraud trend dashboards (feedback reward farming, report brigading, publisher abuse)
- policy reporting exports for D037 governance transparency commitments

This is a platform-operations requirement, not optional UI polish.

**Industry precedent:**

| Project                                | Index Mechanism                                          | Scale          |
| -------------------------------------- | -------------------------------------------------------- | -------------- |
| **Homebrew** (`homebrew-core`)         | Git repo of Ruby formulae; `brew update` = `git pull`    | ~7K packages   |
| **Rust crates.io** (`crates.io-index`) | Git repo of JSON metadata; sparse HTTP fetch added later | ~150K crates   |
| **Winget** (`winget-pkgs`)             | Git repo of YAML manifests; community PRs                | ~5K packages   |
| **Nixpkgs**                            | Git repo of Nix expressions                              | ~100K packages |
| **Scoop** (Windows)                    | Git repo ("buckets") of JSON manifests                   | ~5K packages   |

All of these started with git-as-index and some (crates.io) later augmented with sparse HTTP fetching for performance at scale. The same progression applies here — git index works perfectly for a community of hundreds to low thousands, and can be complemented (not replaced) by a Workshop API when scale demands it.

**Workshop server architecture with P2P:**

```
┌─────────────────────────────────────────────────────┐
│                  Workshop Server                     │
│  ┌─────────────┐  ┌──────────┐  ┌────────────────┐ │
│  │  Metadata    │  │ Tracker  │  │  HTTP Fallback │ │
│  │  (SQLite +   │  │ (BT/WT   │  │  (S3/R2 or     │ │
│  │   FTS5)      │  │  peer     │  │   local disk)  │ │
│  │             │  │  coord)   │  │               │ │
│  └─────────────┘  └──────────┘  └────────────────┘ │
│        ▲               ▲               ▲            │
│        │ search/browse │ announce/     │ GET .icpkg  │
│        │ deps/ratings  │ scrape        │ (fallback)  │
└────────┼───────────────┼───────────────┼────────────┘
         │               │               │
    ┌────┴────┐    ┌─────┴─────┐   ┌─────┴─────┐
    │ ic CLI  │    │  Players  │   │ Seed Box  │
    │ Browser │    │  (seeds)  │   │ (always   │
    └─────────┘    └───────────┘   │  seeds)   │
                                   └───────────┘
```

All three components (metadata, tracker, HTTP fallback) run in the same binary — "just a Rust binary" deployment philosophy. Community self-hosters get the full stack with one executable.

### Rust Implementation

**BitTorrent client library:** The `ic` CLI and game client embed a BitTorrent client. Rust options:
- [`librqbit`](https://github.com/ikatson/rqbit) — pure Rust, async (tokio), actively maintained, supports WebTorrent
- [`cratetorrent`](https://github.com/mandreyel/cratetorrent) — pure Rust, educational focus
- Custom minimal client — only needs download + seed + tracker announce; no DHT, no PEX needed for a controlled Workshop ecosystem

**BitTorrent tracker:** Embeddable in the Workshop server binary. Rust options:
- [`aquatic`](https://github.com/greatest-ape/aquatic) — high-performance Rust tracker
- Custom minimal tracker — HTTP announce/scrape endpoints, peer list management. The Workshop server already has SQLite; peer lists are another table.

**WebTorrent:** `librqbit` has WebTorrent support. The WASM build would use the WebRTC transport.

### Rationale

- **Cost sustainability:** P2P reduces Workshop hosting costs by 90%+. A community project cannot afford CDN bills that scale with popularity. A tracker + seed box for $30-50/month serves unlimited download volume.
- **Fits federation (D030):** P2P is another source in the federated model. The virtual repository queries metadata from remote servers, then downloads content from the swarm — same user experience, different transport.
- **Fits "no single point of failure" (D037):** P2P is inherently resilient. If the Workshop server goes down, peers keep sharing. Content already downloaded is always available.
- **Fits SHA-256 integrity (D030):** P2P needs exactly the integrity verification already designed. Same `manifest.yaml` checksums, same `ic.lock` pinning, same verification on install.
- **Fits WASM target (invariant #10):** WebTorrent enables browser-to-browser P2P. Desktop and browser clients interoperate. No second-class platform.
- **Popular resources get faster:** More downloads → more seeders → faster downloads for everyone. The opposite of CDN economics where popularity increases cost.
- **Self-hosting scales:** Community Workshop servers (D030 federation) benefit from the same P2P economics. A small community server needs only a $5 VPS — the community's players provide the bandwidth.
- **Privacy-responsible:** IP exposure is equivalent to any multiplayer game. HTTP-only mode available for privacy-sensitive users. No additional surveillance beyond standard BitTorrent protocol.
- **Proven technology:** BitTorrent has been distributing large files reliably for 20+ years. Blizzard used it for WoW patches. The protocol is well-understood, well-documented, and well-implemented.

### Alternatives Considered

- **Centralized CDN only** (rejected — financially unsustainable for a donation-funded community project. A popular 500MB mod downloaded 10K times = 5TB = $50-450/month. P2P reduces this to near-zero marginal cost)
- **IPFS** (rejected as primary distribution protocol — slow cold-content discovery, complex setup, ecosystem declining, content pinning is expensive, poor game-quality UX. However, multiple Bitswap protocol design patterns adopted: EWMA peer scoring, per-peer fairness caps, want-have/want-block two-phase discovery, broadcast control, dual WAN/LAN discovery, delegated HTTP routing, batch provider announcements. See competitive landscape table above and research deep dive)
- **Custom P2P protocol** (rejected — massive engineering effort with no advantage over BitTorrent's 20-year-proven protocol)
- **Git LFS** (rejected — 1GB free then paid; designed for source code, not binary asset distribution; no P2P)
- **Steam Workshop only** (rejected — platform lock-in, Steam subsidizes hosting from game sales revenue we don't have, excludes non-Steam/WASM builds)
- **GitHub Releases only** (rejected — works for bootstrap but no search, ratings, dependency resolution, P2P, or lobby auto-download. Adequate interim solution, not long-term architecture)
- **HTTP-only with community mirrors** (rejected — still fragile. Mirrors are one operator away from going offline. P2P is inherently more resilient than any number of mirrors)
- **No git index / custom server from day one** (rejected — premature complexity. A git-hosted index costs $0 and ships with the first playable build. Custom server code can wait until Phase 4-5 when the community is large enough to need search/ratings)

### Phase

- **Phase 0–3:** Git-hosted package index (`workshop-index` repo) + GitHub Releases for `.icpkg` storage. Zero infrastructure cost. Community contributes via PR. Game client fetches `index.yaml` for discovery.
- **Phase 3–4:** Add BitTorrent tracker ($5-10/month VPS). Package manifests gain `torrent` source entries. P2P delivery begins for large packages. Git index remains the discovery layer.
- **Phase 4–5:** Full Workshop server with integrated BitTorrent/WebTorrent tracker, search, ratings, dependency resolution, P2P delivery, HTTP fallback via S3-compatible storage. Git index can coexist or be subsumed.
- **Phase 6a:** Federation (community servers join the P2P swarm), Steam Workshop as additional source, Publisher workflows, and full admin/operator panel + signature/provenance hardening
- **Format recommendations** apply from Phase 0 — all first-party content uses the recommended canonical formats

---

---

