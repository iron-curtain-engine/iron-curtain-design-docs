## D079: Voice-Text Bridge — Speech-to-Text Captions, Text-to-Voice Synthesis, and AI Voice Personas

|                |                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Status**     | Draft                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| **Phase**      | Phase 5 (basic STT captions + basic TTS), Phase 6a (AI voice personas, pluggable backends), Phase 7 (cross-language translation)                                                                                                                                                                                                                                                                                                                                 |
| **Depends on** | D059 (communication system — VoIP, text chat, muting), D034 (SQLite for mute preferences), D052 (community servers — relay forwarding)                                                                                                                                                                                                                                                                                                                              |
| **Driver**     | Xbox shipped party chat STT/TTS in 2021 (Xbox Accessibility Guideline 119). Google Meet shipped AI voice mimicry in 2025. VRChat's mute community (~30% of users choose not to speak) proved the demand for text-based voice alternatives. No RTS has integrated STT/TTS as a first-class communication mode. IC's existing Opus VoIP pipeline (D059) and pluggable LLM/AI backend pattern (`player-flow/llm-setup-guide.md`) make this architecturally natural. |

### Philosophy & Scope Note

> **This decision is Draft — experimental, requires community validation before Accepted.**
>
> **What is proven:** Platform-level STT/TTS is an accessibility standard (Xbox Guideline 119, Sea of Thieves, Forza Horizon 5). Auto-generated captions for voice chat are a top community request across Discord, VRChat, and competitive gaming communities.
>
> **What is experimental:** Per-player AI voice personas and the "shy player pipeline" (type → hear a unique AI voice) are novel as integrated game features. VRChat community mods prove demand exists; Google Meet proves the technology works. But no game ships this as a first-class feature — community validation is needed.
>
> **Philosophy alignment:** This is accessibility-first design (Philosophy Principle 10: "Build with the community"). STT captions and basic TTS directly address the needs of deaf/HoH players and non-verbal players. AI voice personas are experimental flavor on top.

### Decision Capsule (LLM/RAG Summary)

- **Status:** Draft
- **Phase:** Phase 5 (STT captions + basic TTS) → Phase 6a (AI voice personas, pluggable backends) → Phase 7 (cross-language translation)
- **Canonical for:** Voice-to-text transcription, text-to-voice synthesis, AI voice personas, pluggable STT/TTS backends, voice accessibility in multiplayer
- **Scope:** `ic-audio` (STT/TTS processing), `ic-ui` (caption overlay, voice persona settings), `ic-net` (`SynthChatMessage` order routing, `is_synthetic` VoicePacket flag), `ic-game` (backend orchestration)
- **Decision:** IC provides a bidirectional voice-text bridge with two independent formats: (1) auto-generated captions that transcribe incoming voice into text overlays, and (2) a text-to-voice pipeline where typed chat is synthesized into per-player AI voices. Both are opt-in per player, independently controllable, and pluggable (local or cloud backends). Muting operates on three independent channels: voice, synthesized voice, and text.
- **Why:**
  - Accessibility: deaf/HoH players need captions; non-verbal players need a voice alternative (Xbox Guideline 119)
  - Social comfort: shy players, players in noisy environments, or players uncomfortable with their voice can participate in voice-culture lobbies without speaking
  - Cross-language play: STT + translation + TTS enables communication across language barriers (Phase 7)
  - IC's VoIP pipeline (D059 Opus encoding, relay forwarding) and LLM provider pattern already provide the infrastructure
- **Non-goals:** Real-time voice moderation via STT (explicitly rejected in D059 § Alternatives Rejected — compute-intensive, privacy-invasive, unreliable across accents). Voice cloning of other players without consent. Replacing voice chat — this augments it.
- **Out of current scope:** Emotion detection in voice. Voice style transfer (making your voice sound like someone else's in real-time). Real-time voice translation during ranked matches (latency concern).
- **Invariants preserved:** Deterministic sim unaffected (STT is listener-local cosmetic processing; TTS synthesis is client-side and never affects sim state). `SynthChatMessage` is a `PlayerOrder` variant in the order stream (same lane as `ChatMessage` per D059), not a separate message lane. VoIP relay architecture unchanged for Mode B (default); Mode A adds `is_synthetic` bit to `VoicePacket`. Muting model (D059) extended, not replaced.
- **Defaults / UX behavior:** Both STT and TTS are **off by default**. Players opt in via Settings → Communication → Voice-Text Bridge. Default synthesis mode is **receiver-side (Mode B)** — no player hears an AI voice unless they personally enable TTS playback on their own client. Sender-side synthesis (Mode A) requires explicit opt-in from both sender AND receiver (`accept_synthetic_voice` flag). Local models are downloadable model packs (~250MB), not bundled with the base install.
- **Security / Trust impact:** STT runs on the listener's client (the speaker's voice is not transcribed server-side — privacy preserved). Default TTS mode (receiver-side) keeps text on the wire — synthesis is local to the listener. Sender-side TTS (Mode A) adds an `is_synthetic` bit to `VoicePacket` for independent mute routing. `SynthChatMessage` is a new `PlayerOrder` variant (protocol change, replay-visible).
- **Performance impact:** Local STT (Whisper turbo): ~50-200ms per utterance, ~200MB downloadable model pack. Local TTS (Piper/ONNX): <100ms, ~50MB downloadable model pack. Cloud backends: ~75-300ms round-trip. Native platforms: processing on background threads — no frame budget impact. WASM: local models unavailable (no `std::thread`), cloud-only via async fetch on single-threaded executor.
- **Public interfaces / types / commands:** `VoiceTextBridge`, `SttBackend`, `TtsBackend`, `VoicePersonaId` (namespaced `ResourceId`), `VoiceBridgeCapability` (session advertisement), `SynthChatMessage` (order variant), `PlayerMuteState` (extended), `CaptionOverlay`, `/captions`, `/tts`, `/voice-persona`, `/player <name> mute`
- **Affected docs:** `decisions/09g/D059-communication.md` (muting model extension, `VoicePacket` `is_synthetic` flag, `PlayerOrder::SynthChatMessage` variant — protocol version bump), `player-flow/multiplayer.md` (`VoiceBridgeCapability` lobby exchange during ready-up), `player-flow/settings.md` (voice-text bridge settings panel), `player-flow/llm-setup-guide.md` (backend configuration pattern), `formats/save-replay-formats.md` (replay protocol version for `SynthChatMessage`)
- **Keywords:** speech to text, text to speech, STT, TTS, captions, voice accessibility, AI voice, voice persona, voice synthesis, shy player, mute player, cross-language, pluggable backend, Whisper, ElevenLabs, Piper

---

### Problem

IC's communication system (D059) provides text chat channels and relay-forwarded VoIP. Players can mute voice, mute text, or mute both per player. But there is no bridge between the two modalities:

- A deaf player cannot read what a speaking player says (no captions)
- A non-verbal player cannot be "heard" in voice-culture lobbies (no text-to-voice)
- Players who are shy about their voice, in noisy environments, or speaking a different language have no alternative to raw voice chat
- Cross-language teams cannot communicate via voice at all

These gaps are solved by other platforms:
- Xbox Party Chat (2021) ships STT + TTS as platform-level features
- Google Meet (2025) ships AI voice mimicry for translation
- VRChat's community built TTS Voice Wizard because ~30% of users choose to be mute

### Prior Art

| Platform                         | STT (Voice→Text)                | TTS (Text→Voice)                | Personalized Voice                                              | Cross-Language              | Notes                                                                                                                 |
| -------------------------------- | ------------------------------- | ------------------------------- | --------------------------------------------------------------- | --------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| **Xbox Party Chat** (2021)       | Yes — real-time overlay         | Yes — typed text read to party  | No — generic system voices                                      | English (US) only           | Accessibility gold standard. Uses Azure Cognitive Services via PlayFab Party. Ships as platform feature, not per-game |
| **Sea of Thieves**               | Yes — voice→text overlay        | Yes — text→voice for typed chat | No                                                              | English (US) only           | Built on Xbox's PlayFab Party API                                                                                     |
| **Google Meet** (2025)           | Yes — 80+ languages             | Yes — AI speech translation     | **Yes** — synthetic voice mimics speaker's tone, rhythm, pacing | English↔Spanish (expanding) | Uses Gemini AI. The personalized voice feature is the closest precedent to IC's AI voice personas                     |
| **Microsoft Teams** (2025)       | Yes — 34 languages              | Yes — Interpreter Agent         | Partial                                                         | 9 languages                 | Interpreter Agent translates speech in real-time                                                                      |
| **VRChat** (community)           | Yes — TTS Voice Wizard, TaSTT   | Yes — 100+ voices               | **Yes** — voice cloning, character voices                       | 50+ languages               | Community-built tools, not official. Proves demand from mute/shy players                                              |
| **Discord**                      | Not native (bot-only)           | `/tts` (basic)                  | No                                                              | Limited                     | Most-requested accessibility feature, still not native                                                                |
| **ElevenLabs** (API)             | No                              | Yes — 10K+ voices               | **Yes** — clone from 10s audio                                  | 32+ languages               | ~75ms latency. Leading personalized TTS API                                                                           |
| **OpenAI Whisper** (open-source) | Yes — state-of-the-art accuracy | No                              | N/A                                                             | 99 languages                | Open-source, runs locally. Turbo model: 216x real-time speed. Privacy-safe (no cloud). ~200MB model                   |

### Decision

IC implements a **Voice-Text Bridge** — a bidirectional, opt-in system with two independent formats and pluggable backends.

---

#### Format 1: Auto-Generated Captions (STT — Voice→Text)

Incoming voice chat is transcribed into text and displayed as a caption overlay on the listener's screen. **Processing happens entirely on the listener's client** — the speaker's voice is never transcribed server-side.

##### UX Flow

1. Listener enables captions in Settings → Communication → Voice-Text Bridge → Captions: On
2. When a teammate speaks, the listener's STT backend transcribes the Opus audio stream
3. Transcribed text appears as a caption overlay at the bottom of the screen (configurable position)
4. Captions are attributed to the speaker (player name + faction color)
5. Captions fade after 5 seconds (configurable)

##### Caption Overlay

```
┌──────────────────────────────────────────────────────┐
│                    GAME VIEW                          │
│                                                      │
│                                                      │
│                                                      │
│  ┌────────────────────────────────────────────┐      │
│  │ [Alice] Attack the north bridge             │      │
│  │ [Bob]   I'll send tanks to support          │      │
│  └────────────────────────────────────────────┘      │
│  ▲ Caption overlay (position/size configurable)       │
└──────────────────────────────────────────────────────┘
```

##### Multi-Language Captions (Phase 7)

When cross-language translation is enabled, captions are displayed in the listener's preferred language:

1. STT transcribes voice → source-language text
2. Translation engine translates source text → listener's language
3. Caption shows translated text with a language indicator **and machine-translation trust label** (per D068 § Machine-Translated Content Labeling): `[Alice 🇫🇷→🇬🇧 ⚙️MT] Attack the north bridge`

The `⚙️MT` (machine-translated) label follows the same pattern D068 uses for machine-translated UI strings and mod descriptions — the player must always know when text has been machine-translated rather than human-authored. The label is non-dismissible (always visible when translation is active).

##### Caption Interaction with Chat

Captions are **not injected into the text chat channel** — they appear only in the caption overlay. D059 treats the chat panel as deterministic order-stream data preserved for replay/review (`D059-overview-text-chat-voip-core.md` § Chat Architecture). STT transcripts are listener-local, non-deterministic artifacts (different listeners may get different transcriptions of the same speech) and must never be mixed into the canonical chat stream.

If the player wants scrollback review of transcripts, they are displayed in a **separate transcript panel** (a secondary tab or collapsible section below the chat panel), visually distinct from chat:

- Transcript entries are tagged with a `🎤 STT` prefix and rendered in italic or a muted color
- They are **not** stored in the replay's chat log (they are ephemeral, listener-local)
- They are **not** visible to other players (each listener's transcription is private to their client)

---

#### Format 2: Text-to-Voice Pipeline (TTS — Text→Voice)

A player types in text chat, and the receiving player hears it as synthesized speech. The sender's text is transmitted as a chat order; the receiver's client synthesizes it into audio locally.

##### Two Synthesis Modes

**Mode B — Receiver-Side Synthesis (default):**

The flow: **Player A configures voice parameters → sends chat message → Player B has TTS enabled → Player B's TTS engine reads Player A's text using Player A's voice parameters, after preprocessing the text for the engine.**

1. **Player A** configures their voice persona (voice model, pitch, speed, style) in Settings. These parameters are exchanged with all players during the **lobby waiting room** (ready-up phase) via `VoiceBridgeCapability` — before the game starts. The receiver's client can pre-load the correct TTS voice model during the loading screen
2. **Player A** types a message in text chat. Their client sends a `SynthChatMessage` order containing the text + Player A's `persona_id`
3. **Player B** receives the `SynthChatMessage`. Player B's client checks: does Player B have TTS playback enabled? If no → display as normal text, done. If yes → continue
4. Player B's client looks up Player A's voice parameters (received earlier via `VoiceBridgeCapability`): persona ID, pitch, speed, style
5. Player B's TTS engine preprocesses the text (rule-based + spell correction — see § TTS Text Preprocessing) and synthesizes it using **Player A's configured voice parameters** — not Player B's own
6. Player B hears the message spoken in Player A's chosen voice. Player B controls *whether* to hear TTS, but the *voice characteristics* are Player A's

This is the default because it respects the stated UX rule: "no player hears an AI voice unless they personally enable TTS playback." The sender configures *how* they sound; the receiver decides *whether* to listen.

**Mode A — Sender-Side Synthesis (opt-in, requires receiver consent):**
1. Player types a message in text chat
2. Player's client synthesizes the text into audio using their configured TTS backend and voice persona
3. The synthesized Opus audio is transmitted via the VoIP relay lane with `is_synthetic: true` in the voice packet (see § Voice Packet Extension below)
4. Receivers who have "Accept synthesized voice" enabled hear it as voice
5. Receivers who have NOT opted in, or who have the sender's synth muted, silently drop the packet — the message still arrives as text via the parallel `SynthChatMessage` order

Mode A is opt-in for **both** sender and receiver. The sender must enable "Send as voice" and the receiver must enable "Accept synthesized voice." This prevents unsolicited AI voice injection into lobbies.

**D059 competitive voice restrictions apply fully to synthetic voice.** Mode A synthesized packets inherit all `VoiceTarget` routing rules from D059:
- **Ranked:** `VoiceTarget::All` is disabled — synthetic voice is team-only, same as real voice (D059 § Ranked voice channel restrictions)
- **Observers:** Synthetic voice from observers is never forwarded to players during live games (D059 § Anti-coaching)
- **Tournament:** Organizer controls whether synthetic voice is allowed via `TournamentConfig`
- Synthetic voice packets are subject to the same relay mute/moderation pipeline as real voice (admin mute, abuse penalties, etc.)

Synthetic voice does **not** bypass these restrictions. The relay enforces routing rules identically for `is_synthetic: true` and `is_synthetic: false` packets — the flag is used only for per-player synth muting, not for routing policy.

**Trade-offs:**

| Aspect            | Sender-Side (Mode A)                                             | Receiver-Side (Mode B, default)                   |
| ----------------- | ---------------------------------------------------------------- | ------------------------------------------------- |
| Network           | Uses VoIP bandwidth (Opus audio) + text                          | Uses text bandwidth only (minimal)                |
| Latency           | Sender's TTS latency + network                                   | Network + receiver's TTS latency                  |
| Voice consistency | All opted-in receivers hear the same voice                       | Each receiver uses their own TTS backend/quality  |
| Privacy           | Synthesized audio leaves sender's machine                        | Only text leaves sender's machine (better)        |
| Opt-in guarantee  | Requires receiver consent flag                                   | Preserved by design (receiver controls synthesis) |
| Replay            | Heard as voice in replay (if voice-in-replay enabled)            | Text in order stream + voice params in replay metadata (see below) |
| Muting            | Requires `is_synthetic` packet flag for independent synth muting | Receiver controls locally — trivial to mute       |

##### Replay Voice Reproduction (Mode B)

Mode B records `SynthChatMessage` orders (with `persona_id`) in the replay's tick order stream, but the sender's `VoiceParams` (pitch, speed) are exchanged at lobby time, not per-message. To enable replay viewers to synthesize the same voice:

- All players' `VoiceBridgeCapability` data is stored in the **replay metadata JSON** (alongside player names, factions, and other session info). This adds ~50 bytes per player — negligible.
- On replay playback, the viewer reads the capability set from metadata, resolves each player's persona + voice params, and can synthesize `SynthChatMessage` text in the correct voice if the viewer has TTS enabled.
- If the replay viewer does not have TTS enabled or lacks the persona's model, `SynthChatMessage` orders display as normal text — the text is always preserved regardless of TTS capability.

```json
{
  "players": [
    { "slot": 0, "name": "Alice", "faction": "allies", ... },
    { "slot": 1, "name": "Bob", "faction": "soviet", ... }
  ],
  "voice_bridge_capabilities": [
    { "slot": 0, "tts_enabled": true, "persona_id": "official/commander-alpha", "pitch": 0.9, "speed": 1.1 },
    { "slot": 1, "tts_enabled": false, "persona_id": null, "pitch": 1.0, "speed": 1.0 }
  ]
}
```

##### Chat Order Extension

The current `PlayerOrder::ChatMessage { channel, text }` has no metadata field. D079 adds a new order variant for TTS-eligible messages:

```rust
/// Extension to PlayerOrder for voice-text bridge (D079).
/// When the sender has TTS enabled, SynthChatMessage is sent INSTEAD OF
/// ChatMessage (not alongside — one canonical order per message, no
/// duplicates in the order stream or replay). Receivers who don't have
/// TTS enabled display the text field as a normal chat message and
/// ignore persona_id.
pub enum PlayerOrder {
    // ... existing variants ...
    ChatMessage { channel: ChatChannel, text: String },

    /// TTS-eligible chat message. Same as ChatMessage but includes
    /// the sender's voice persona ID for receiver-side synthesis.
    SynthChatMessage {
        channel: ChatChannel,
        text: String,
        /// Sender's voice persona ID (namespaced ResourceId).
        /// Receivers use this to select the TTS voice for synthesis.
        /// e.g., "official/commander-alpha", "community/modx-voices/tactical".
        /// "local/*" IDs trigger fallback to a built-in persona on receivers
        /// who don't have the local definition.
        persona_id: Option<VoicePersonaId>,
    },
}
```

This is a **replay-visible protocol change requiring a protocol version bump.** `SynthChatMessage` is a new `PlayerOrder` discriminant. Per `api-misuse-defense.md` § O4, unknown `PlayerOrder` discriminants are deserialization errors — not silently skipped — and protocol version mismatch terminates the handshake before orders flow. There is no graceful mixed-version decoding. Clients must match protocol versions to play together. The version bump is reflected in the replay header's `version` field and the wire format's protocol version header.

##### Voice Packet Extension (Mode A only)

For sender-side synthesis, the `VoicePacket` struct (D059) gains a `is_synthetic` flag to enable independent muting:

```rust
pub struct VoicePacket {
    pub speaker: PlayerId,
    pub sequence: u32,
    pub is_synthetic: bool,    // D079: true if this packet is TTS-synthesized, not real voice
    // ... existing fields ...
}
```

This **is** a protocol change to D059's voice packet format — the `is_synthetic` flag occupies one bit in the packet header. Relay routing uses it: if a receiver has `synth_voice_muted` for this speaker, the relay skips forwarding synthetic packets (same bandwidth-saving pattern as D059's mute hint). This change is part of the same protocol version bump as `SynthChatMessage` — voice packets are versioned alongside the order protocol. No mixed-version voice decoding.

##### TTS Text Preprocessing (Optional)

Raw chat text often contains typos, abbreviations, gaming slang, special symbols, emoji, and non-standard grammar that TTS engines handle poorly — producing garbled, robotic, or nonsensical speech. A **preprocessing step** normalizes the text for the TTS engine before synthesis, while leaving the original chat text untouched for display.

**What it fixes:**

| Input Pattern                           | Problem for TTS                                                 | Preprocessed Output                         |
| --------------------------------------- | --------------------------------------------------------------- | ------------------------------------------- |
| `atk north w/ tanks asap`               | Abbreviations read literally ("ay tee kay", "double-you slash") | "Attack north with tanks, ASAP"             |
| `gg wp`                                 | Read as individual letters                                      | "Good game, well played"                    |
| `lol ur base is ded 😂`                  | Emoji read as unicode name, slang mispronounced                 | "Ha, your base is dead"                     |
| `need $$ for mamoth tank`               | Symbols read literally, typo mispronounced                      | "Need money for mammoth tank"               |
| `!!!!! HELP HELP`                       | Excessive punctuation causes stutter/pause spam                 | "Help! Help!"                               |
| `сука блять` (Cyrillic in English chat) | TTS engine for English can't pronounce Cyrillic                 | Transliterated or skipped with notification |
| `<script>alert('xss')</script>`         | Not a TTS issue, but sanitization                               | Stripped (sanitized before display anyway)  |

**Implementation tiers (lightest to heaviest):**

| Tier                                         | Approach                                                                                                                                                                                                                                | Latency  | Size                   | Quality                       | Phase    |
| -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ---------------------- | ----------------------------- | -------- |
| **1. Rule-based** (default)                  | YAML dictionary of abbreviations/slang → expansions. Regex symbol stripping. Emoji → description mapping. Punctuation collapsing. WFST-style number/currency/unit expansion ("$5k" → "five thousand dollars"). Moddable per game module | <1ms     | ~50KB dictionary       | Good for common cases         | Phase 5  |
| **2. Spell correction** (optional)           | SymSpell algorithm (symmetric delete, edit distance 2). Pre-computed dictionary of valid words. 1M× faster than Norvig's approach, O(1) lookup. Catches typos like "mamoth" → "mammoth", "atack" → "attack". Language-independent       | <1ms     | ~5MB dictionary        | Excellent for typos           | Phase 5  |
| **3. Statistical normalization** (optional)  | Ekphrasis-style word segmentation + normalization using word frequency statistics (from gaming corpora). Handles hashtag splitting, elongation normalization ("nooooo" → "no"), social-media-style text. No neural network              | ~5ms     | ~20MB frequency tables | Good for slang/shorthand      | Phase 6a |
| **4. WFST text normalization** (optional)    | Weighted Finite State Transducer grammars (NeMo-style). Deterministic, rule-compiled. Handles complex number formats, dates, addresses, unit conversions. Production-grade TTS preprocessing used by NVIDIA and Google                  | <5ms     | ~10MB compiled grammar | Excellent for structured text | Phase 6a |
| **5. LLM/SLM-enhanced** (optional, advanced) | Small local model or cloud LLM for context-aware normalization. Handles novel abbreviations, ambiguous intent, mixed-language input                                                                                                     | 50-500ms | ~500MB+ (local)        | Best for edge cases           | Phase 7  |

**Default stack:** Tier 1 (rules) + Tier 2 (spell correction) ship in Phase 5. Both are <1ms, zero neural dependencies, fully offline. Tiers 3-5 are opt-in upgrades for players who want higher quality preprocessing.

**Rule-based dictionary (YAML, moddable):**

```yaml
# tts-preprocessing/en.yaml — shipped with IC, moddable per game module
abbreviations:
  gg: "good game"
  wp: "well played"
  glhf: "good luck, have fun"
  asap: "as soon as possible"
  atk: "attack"
  def: "defend"
  w/: "with"
  ur: "your"
  u: "you"
  plz: "please"
  thx: "thanks"
  omw: "on my way"
  brb: "be right back"
  afk: "away from keyboard"
  ez: "easy"
  imo: "in my opinion"
  tbh: "to be honest"
  nvm: "never mind"

# Game-specific terms (ra1 module)
game_terms:
  mamoth: "mammoth"        # common typo
  mig: "MiG"               # pronunciation hint
  mcv: "M.C.V."            # spell out abbreviation
  apc: "A.P.C."
  gdi: "G.D.I."
  nod: "Nod"

emoji_map:
  "😂": "(laughing)"
  "💀": "(skull)"
  "🔥": "(fire)"
  "👍": "(thumbs up)"
  "❤️": "(heart)"

rules:
  collapse_repeated_punctuation: true   # "!!!!" → "!"
  collapse_repeated_letters: true       # "nooooo" → "nooo" (preserve some emphasis)
  max_repeated_chars: 3
  strip_markdown: true                  # remove **bold**, *italic* markers
  strip_html_tags: true                 # sanitization
```

**Tier 5 — LLM/SLM-enhanced preprocessing (Phase 7, optional, advanced):**

For players who have a local LLM configured (same provider pattern as D047/`llm-setup-guide.md`), text can pass through a context-aware normalization prompt. This is the heaviest tier and is **not recommended as the default** — Tiers 1-2 cover the vast majority of cases. Use only when dealing with highly novel slang, mixed-language input, or ambiguous abbreviations that rule-based and statistical tiers can't handle.

```
System: You are a text normalizer for a text-to-speech engine in a military RTS game.
Convert the input chat message into clean, natural spoken English.
Fix typos, expand abbreviations, remove excessive punctuation, describe relevant emoji.
Keep it brief. Do not add content. Do not censor. Output only the normalized text.

Input: "atk north w/ tanks asap, ur base is ded lol 😂"
Output: "Attack north with tanks, ASAP. Your base is dead. Ha!"
```

**Settings:**

```yaml
[voice_text_bridge.tts.preprocessing]
enabled = true                           # on by default when TTS is enabled
tier = "rules+spell"                     # "rules" | "rules+spell" (default) | "statistical" | "wfst" | "llm" | "disabled"
dictionary = "tts-preprocessing/en.yaml" # Tier 1 rules dictionary (moddable per game module / language)
spell_dictionary = "tts-preprocessing/en-spell.txt"  # Tier 2 SymSpell word frequency list
llm_provider = ""                        # only used when tier = "llm"
```

**The original chat text is never modified.** Preprocessing produces a parallel normalized string that is fed to the TTS engine only. Other players see the original text in the chat panel. The sender sees their original text too — the preprocessing is invisible to everyone except the TTS engine.

---

#### AI Voice Personas (Phase 6a)

Each player can be assigned (or choose) a unique AI-generated voice persona that is used for their TTS synthesis. This ensures that when multiple players use text-to-voice, each sounds distinct — not the same generic robot voice.

##### Persona Assignment

```yaml
# Voice persona definition (stored in <data_dir>/voice-personas/ or shipped in packs)
voice_persona:
  id: "official/commander-alpha"      # namespaced ResourceId
  display_name: "Commander Alpha"
  pitch: 0.9                     # relative pitch adjustment (0.5 = deep, 1.0 = neutral, 1.5 = high)
  speed: 1.1                     # speaking rate (0.5 = slow, 1.0 = normal, 1.5 = fast)
  language: en-US
  tts_model: "piper:en_US-lessac-medium"  # local model reference
  # OR: tts_model: "elevenlabs:voice_id_abc123"  # cloud model reference
```

##### Unified Voice Identity (ArmA Pattern)

Inspired by ArmA's player voice system — where players pick a voice type and pitch that is used for all radio commands. In IC, the voice persona is a **unified voice identity**: whenever *any* text is synthesized for a player (typed chat, chat wheel phrase, beacon alert, quick command), the TTS engine uses that player's configured voice parameters.

The key insight: the TTS engine doesn't care whether the input text is a player's typed message or a system-generated "Affirmative" string. It's all text → voice. The player's persona parameters (voice model, pitch, speed, style) are applied uniformly to all synthesis for that player.

| Text Source | Example Text | TTS Treatment |
|-------------|-------------|---------------|
| Player types in chat | "atk north with tanks" | Synthesized with player's persona voice |
| Chat wheel phrase (D059) | "Affirmative" / "Need backup" | Same persona voice — the phrase text is just fed to the TTS engine |
| Beacon alert (D059) | "Enemy spotted at north bridge" | Same persona voice |
| Quick command | "Moving out" / "Understood" | Same persona voice |

This means each player in a match sounds distinct — not because they picked from pre-recorded audio samples, but because the TTS engine produces different-sounding output based on each player's voice/pitch/speed configuration. Two players saying "Affirmative" sound different because their personas have different parameters.

**Player-facing settings (Settings → Communication → Voice Persona):**

| Setting | Range | Default | Description |
|---------|-------|---------|-------------|
| Voice | Built-in library or custom | Auto-assigned | Base TTS model/voice — the "raw material" the engine synthesizes from |
| Pitch | 0.5 – 1.5 slider | 1.0 (neutral) | Lower = deeper, higher = lighter |
| Speed | 0.5 – 1.5 slider | 1.0 (normal) | Speaking rate |
| Preview | [Play Sample] button | — | Hear your persona say a sample phrase before committing |

**Note on style/post-processing:** D059's voice effects rule is that the **listener** controls audio post-processing (radio filter, reverb, clean audio — see `D059-voip-effects-ecs.md` § Voice Effects). The sender controls voice *identity* (which voice, pitch, speed); the receiver controls *how they hear it* (effects, spatial audio, filter). Style post-processing is therefore **not** part of `VoiceParams` — it stays receiver-side, consistent with D059.

These parameters are stored in the player's local settings and advertised as part of the `VoiceBridgeCapability` session advertisement (see § Persistence Model). The receiver's TTS engine uses them when synthesizing any text attributed to that player.

##### Persona Sources

| Source              | How It Works                                                                                                                                               | Phase    |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| **Built-in voices** | IC ships 8-12 built-in personas per language (varied gender, age, style). Assigned round-robin to players who don't choose one                             | Phase 5  |
| **Player-selected** | Player picks from built-in library or configures a custom persona in Settings                                                                              | Phase 6a |
| **Faction-themed**  | Allied voices sound Western/NATO; Soviet voices sound Eastern European. Automatic based on faction                                                         | Phase 6a |
| **Workshop voices** | Community-created voice persona packs (D050). Must pass content moderation (D037)                                                                          | Phase 6a |
| **Cloud-cloned**    | Player uses a cloud TTS API (ElevenLabs) to create a voice from a short recording. The cloud voice ID is stored in the player's local settings. **Receiver-side limitation:** cloud-cloned voices only work in sender-side synthesis (Mode A), because the receiver cannot access the sender's cloud API credentials. In receiver-side synthesis (Mode B, default), cloud-cloned personas **fall back to the nearest matching built-in persona** (matched by pitch/speed/style parameters). The sender hears their cloud voice locally in preview; teammates hear the built-in fallback unless Mode A is used | Phase 6a |

##### Persona Transmission

When using sender-side synthesis (Mode A), the synthesized audio already carries the persona's characteristics — no additional metadata needed beyond `is_synthetic` in the voice packet. When using receiver-side synthesis (Mode B, default), the sender's `persona_id` is included in the `SynthChatMessage` order (see § Chat Order Extension) so the receiver's client can select the correct TTS voice.

---

#### Pluggable Backend Architecture

STT and TTS backends are independently pluggable, following the same provider pattern as the LLM setup guide (`player-flow/llm-setup-guide.md`).

##### Persistence Model — What Lives Where

Three systems, three purposes — no overlap:

| Data                                                                                    | Home                                             | Why                                                                                                                              |
| --------------------------------------------------------------------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| **Backend config** (which STT/TTS engine, API keys, model paths, caption position)      | `settings.toml` § `[voice_text_bridge]`          | Local machine preferences. Not shared. Same pattern as `[llm]` and `[audio]` settings                                            |
| **Voice persona handle** (selected persona ID — a namespaced `ResourceId`, not the full definition) | `settings.toml` + session capability advertisement | Local preference stored in settings. Transmitted to teammates via lobby capability exchange (not via D053 profile — see rationale below). ~40 bytes |
| **Custom persona definitions** (full TTS model refs, pitch/speed/style, cloud vendor refs) | Local data dir (`<data_dir>/voice-personas/`) | Too large and too private for the shared profile. D053 assumes ~2KB profile responses. Custom blobs may include vendor API refs that shouldn't be advertised |
| **Per-player mute state** (voice/synth/text mute decisions)                             | Local SQLite (D034), keyed by Ed25519 public key | Local preference — same pattern as D059's existing mute persistence. Not synced                                                  |

**Settings IA note:** The current settings panel (`settings.md`) has **Audio** and **Social** tabs but no **Communication** tab. Voice-text bridge settings should be added as a new **Communication** sub-section under **Social** (where voice/chat settings naturally belong), or as a new top-level tab if the surface area warrants it. This is a `settings.md` update deferred until D079 moves to Accepted.

**Voice persona is NOT in D053 player profile.** D053 requires every profile field to have visibility controls (D053 § Design Principles: "Privacy by default. Every profile field has visibility controls"). But teammates **must** always receive the persona ID for receiver-side TTS to work — a privacy toggle would break synthesis. This makes it a poor fit for the privacy-controlled profile model.

Instead, the voice persona ID is transmitted as a **session capability advertisement** during lobby join — the same mechanism used for engine version, mod fingerprint, and other per-session metadata. It's ephemeral (valid for this session only), always visible to participants, and not part of the persistent public profile.

**Namespaced `ResourceId` format:** Persona IDs follow the project's standard namespaced resource pattern (D068, D062):

```
official/commander-alpha          # built-in persona shipped with IC
official/soviet-officer           # built-in faction-themed persona
community/modx-custom-voices/tactical  # Workshop persona pack
local/my-custom-voice             # locally-defined persona (not resolvable by others — fallback to built-in)
```

Bare strings like `"custom_42"` are rejected — the namespace prefix is mandatory. This prevents collisions across packs/machines and makes resolution unambiguous: `official/` resolves from built-in data, `community/` resolves from installed Workshop packs, `local/` triggers fallback on receivers who don't have it.

```rust
/// Voice persona ID — namespaced ResourceId, not a freeform string.
/// Transmitted in SynthChatMessage.persona_id and lobby capability exchange.
/// NOT part of D053 PlayerProfile (see rationale above).
pub struct VoicePersonaId(pub ResourceId);  // e.g., "official/commander-alpha"

/// Lobby/waiting room capability advertisement for voice-text bridge.
/// Exchanged during the lobby ready-up phase (before the game starts),
/// alongside engine version, mod fingerprint, and other session metadata.
/// Each player's voice configuration is available to all participants
/// before the first tick — the receiver's TTS engine can pre-load the
/// correct voice model during the loading screen.
/// See player-flow/multiplayer.md § Lobby for the ready-up flow.
pub struct VoiceBridgeCapability {
    /// Whether this player sends SynthChatMessage orders (has TTS enabled).
    pub tts_enabled: bool,
    /// The player's selected voice persona (namespaced ResourceId).
    /// Determines which base TTS model/voice the receiver's engine loads.
    pub persona_id: Option<VoicePersonaId>,
    /// The player's voice parameters — applied ON TOP of the base persona
    /// by the receiver's TTS engine. These are the sender's configured
    /// pitch/speed/style, not the receiver's preferences.
    pub voice_params: Option<VoiceParams>,
}

/// Player-configured voice parameters. Advertised to teammates via
/// VoiceBridgeCapability so the receiver's TTS engine can reproduce
/// the sender's chosen voice characteristics.
pub struct VoiceParams {
    pub pitch: f32,     // 0.5 (deep) – 1.5 (high), default 1.0
    pub speed: f32,     // 0.5 (slow) – 1.5 (fast), default 1.0
    // NOTE: No `style` field. Post-processing (radio filter, reverb, etc.)
    // is receiver-controlled per D059 § Voice Effects. The sender controls
    // voice identity (model + pitch + speed); the receiver controls effects.
}
```

Full custom persona definitions remain in `<data_dir>/voice-personas/` (local data, shareable via Workshop packs). `accept_synthetic_voice` remains in `settings.toml` (local receive preference).

**Reconnect and late-observer rule:** `VoiceBridgeCapability` is exchanged at lobby ready-up, but players may reconnect mid-match (D059 § desync recovery) and observers may join after launch (D059 § spectator). For these cases, the relay includes all active players' `VoiceBridgeCapability` data in the reconnection snapshot metadata — the same mechanism used for player names, factions, and other session info that a reconnecting client needs to reconstruct the game state. Late-joining observers receive the capability set as part of the mid-game join handshake. No separate resend protocol is needed — it piggybacks on existing reconnection infrastructure.

##### Settings Configuration

```toml
# settings.toml — local machine config (NOT social/profile state)
[voice_text_bridge]
enabled = false                          # master toggle (off by default)

[voice_text_bridge.stt]
enabled = false                          # captions off by default
backend = "local"                        # "local" | "cloud" | "disabled"
local_model = "whisper-turbo"            # model pack name (downloaded via D068, not bundled)
cloud_provider = ""                      # "azure" | "google" | "deepgram" (requires API key)
cloud_api_key_ref = ""                   # reference to credential in ic-paths keystore
language = "auto"                        # auto-detect or explicit language code
caption_position = "bottom"              # "bottom" | "top" | "left" | "right"
caption_duration_sec = 5.0
show_transcript_panel = false            # show separate transcript panel (scrollback review, never in chat log)

[voice_text_bridge.tts]
enabled = false                          # text-to-voice off by default
backend = "local"                        # "local" | "cloud" | "disabled"
local_model = "piper:en_US-lessac-medium"  # model pack name (downloaded via D068)
cloud_provider = ""                      # "elevenlabs" | "azure" | "google"
cloud_api_key_ref = ""
synthesis_mode = "receiver"              # "receiver" (Mode B, default) | "sender" (Mode A, requires receiver consent)
accept_synthetic_voice = false           # opt-in to hear Mode A synthesized voice from other players

[voice_text_bridge.tts.preprocessing]
enabled = true                           # on by default when TTS is enabled
tier = "rules+spell"                     # "rules" | "rules+spell" (default) | "statistical" | "wfst" | "llm" | "disabled"
dictionary = "tts-preprocessing/en.yaml"
spell_dictionary = "tts-preprocessing/en-spell.txt"

[voice_text_bridge.translation]
enabled = false                          # Phase 7
target_language = ""                     # translate captions to this language
```

##### Backend Options

| Backend                      | STT | TTS | Latency    | Privacy            | Quality   | Delivery                         |
| ---------------------------- | --- | --- | ---------- | ------------------ | --------- | -------------------------------- |
| **Local Whisper (ONNX)**     | Yes | No  | ~100-500ms | Full (on-device)   | High      | Downloadable model pack (~200MB) |
| **Local Piper**              | No  | Yes | <100ms     | Full (on-device)   | Medium    | Downloadable model pack (~50MB)  |
| **Azure Cognitive Services** | Yes | Yes | ~200-400ms | Cloud (Microsoft)  | Very high | API key (no local model)         |
| **Google Cloud Speech/TTS**  | Yes | Yes | ~200-400ms | Cloud (Google)     | Very high | API key (no local model)         |
| **ElevenLabs**               | No  | Yes | ~75ms      | Cloud (ElevenLabs) | Excellent | API key (no local model)         |
| **Deepgram**                 | Yes | No  | ~100-300ms | Cloud (Deepgram)   | Very high | API key (no local model)         |

**Model delivery:** Local STT/TTS models are **not bundled with the base IC install** — they are downloadable model packs following the same pattern as LLM model packs (`llm-setup-guide.md` § Quickest Start) and optional content packs (D068 selective install, D069 install wizard). When a player enables voice-text bridge for the first time:

1. Settings → Communication → Voice-Text Bridge → Enable
2. IC prompts: "Voice-text bridge requires a model pack (~250MB for STT + TTS). Download now?"
3. Model pack downloaded from Workshop (D050) or IC CDN — installed into `<data_dir>/models/voice-bridge/`
4. Done — local STT/TTS active, works offline from this point

This respects the project's install philosophy: the base binary is small; large optional content (models, HD assets, campaigns) is downloaded on demand via managed packs. Players who only use cloud backends (API key) need no local models at all.

**Default experience (after model download):** Local Whisper for STT + Local Piper for TTS. Works offline, no cloud dependency, acceptable quality. Players who want higher quality or personalized voices configure a cloud provider via the same setup flow as LLM providers.

---

#### Extended Muting Model

D059's existing mute model supports per-player voice and text muting. D079 extends this with a third independent channel — synthesized voice:

```rust
/// Extended per-player mute state (D059 + D079).
pub struct PlayerMuteState {
    /// Mute the player's real voice (Opus VoIP stream)
    pub voice_muted: bool,
    /// Mute the player's text chat messages
    pub text_muted: bool,
    /// Mute the player's synthesized voice (TTS playback of their text)
    /// Independent from voice_muted — a player might mute someone's real
    /// voice but still hear their typed messages as synthesized speech,
    /// or vice versa.
    pub synth_voice_muted: bool,
}
```

**Mute combinations and their effects:**

| voice_muted | text_muted | synth_voice_muted | Effect                                                     |
| ----------- | ---------- | ----------------- | ---------------------------------------------------------- |
| false       | false      | false             | Hear everything (real voice + text + synth)                |
| true        | false      | false             | No real voice; see text; hear synth of their text          |
| false       | true       | false             | Hear real voice; no text; no synth (nothing to synthesize) |
| false       | false      | true              | Hear real voice; see text; no synth playback               |
| true        | false      | true              | No voice at all; see text only                             |
| true        | true       | true              | Fully muted — no communication received                    |

**Console commands (D058):**

> **Namespace note:** `/mute` is already overloaded across three systems — D058 uses `/mute <player>` for admin chat mute and `/mute [master|music|sfx|voice]` for audio mixing; D059 uses `/mute <player>` for local player mute. D079 does **not** add another `/mute` variant. Instead, it uses the `/player` namespace for per-player communication control, keeping `/mute` for its existing admin and audio uses.

```
/captions on|off                                -- toggle STT captions
/captions language <code>                       -- set caption language (or 'auto')
/tts on|off                                     -- toggle TTS for your outgoing text (mid-match: toggles SynthChatMessage emission)
/tts mode sender|receiver                       -- set synthesis mode (lobby only — locked after match start)
/voice-persona list                             -- show available personas
/voice-persona set <id>                         -- set your voice persona (lobby only — locked after match start)
/voice-persona preview <id>                     -- hear a sample of a persona (lobby only)
/player <name> mute voice|text|synth|all        -- mute specific channels for a player
/player <name> unmute voice|text|synth|all      -- unmute specific channels
/player <name> mute-status                      -- show current mute state for a player
```

D058 currently has `/players` (list connected players) but no `/player <name> ...` command family. D079 **introduces** the `/player <name>` namespace for per-player communication control, avoiding further overload of `/mute` (which D058 uses for both admin chat mute and audio mixing). D058 should be updated to formalize this namespace when D079 moves to Accepted. The existing D059 `/mute <player>` shorthand (which mutes voice + text) continues to work as a convenience alias for `/player <name> mute all`.

**Session-lock rule:** Voice persona and synthesis mode are **locked after match start**. `VoiceBridgeCapability` is exchanged once during lobby ready-up — there is no runtime capability-update path. Changing persona mid-match would require re-broadcasting to all participants and potentially re-loading TTS models on every receiver, which is disruptive and complex. `/tts on|off` remains toggleable mid-match (it only controls whether the local client emits `SynthChatMessage` orders — no capability re-broadcast needed). `/voice-persona set` and `/tts mode` are rejected with a message after match start: "Voice persona and synthesis mode can only be changed in the lobby."

---

#### Implementation Architecture

##### Processing Pipeline

```
CAPTION PATH (speaker → caption on listener's screen):
  Mic → Opus encode → Relay → Opus decode → [Listener's STT] → Caption overlay
                                                                     ↓
                                                        (optional) Chat log

TTS PATH (typist → voice on listener's speakers):
  Keyboard → Text message → [Preprocess] → [Sender's TTS] → Opus encode → Relay → Speaker
       (Mode A: sender-side)     ↑                                              ↓
                          Rule-based or                                   Listener hears voice
                          LLM normalize
                          (original text
                           untouched in chat)
  OR:

  Keyboard → Text message → Relay → [Preprocess] → [Listener's TTS] → Speaker
       (Mode B: receiver-side)            ↑                           ↓
                                   Rule-based or               Listener hears voice
                                   LLM normalize
```

##### Threading Model

STT and TTS run on **background threads** on native platforms — never on the game loop or render thread:

```
Native (Windows/macOS/Linux/Steam Deck):
  Main thread (GameLoop):  Orders, sim, render — unaffected
  Audio thread:            Opus encode/decode, jitter buffer (existing D059)
  STT thread:              Whisper inference (background, feeds captions to UI via channel)
  TTS thread:              Piper/cloud synthesis (background, feeds Opus frames to audio thread)

Browser (WASM):
  Single-threaded executor (crate-graph.md § WASM: no std::thread).
  Local STT/TTS models are NOT available (Whisper/Piper require background threads).
  Cloud-only backends work via async fetch (non-blocking HTTP calls on the main executor).
  If no cloud backend is configured, voice-text bridge is disabled on WASM with a
  settings note: "Local voice models require the native app. Use a cloud provider
  for browser play."
```

**Per-platform policy:**

| Platform | Local STT/TTS | Cloud STT/TTS | Threading |
|----------|---------------|---------------|-----------|
| Windows, macOS, Linux, Steam Deck | Yes (background threads) | Yes (async HTTP on background thread) | `std::thread` |
| Mobile (iOS/Android) | Yes (background threads, but battery/thermal concern — default off, cloud preferred) | Yes | Platform threads |
| Browser (WASM) | **No** (no `std::thread`, models too large for browser) | Yes (async fetch on single-threaded executor) | Single-threaded async |

##### Relay Impact

**Two protocol changes (require protocol version bump — no mixed-version decoding per `api-misuse-defense.md` § O4):**

1. **`SynthChatMessage` order variant** — a new `PlayerOrder` variant alongside `ChatMessage`. Adds `persona_id: Option<VoicePersonaId>` (namespaced `ResourceId`). Relay routes it identically to `ChatMessage` (same lane, same filtering).

2. **`is_synthetic` bit in `VoicePacket`** (Mode A only) — one bit in the packet header. Relay reads it for mute-hint routing: if a receiver has `synth_voice_muted` for this speaker, the relay skips forwarding (same bandwidth-saving pattern as D059's per-player mute hint). Older clients that don't understand the flag treat the packet as normal voice.

For the default Mode B (receiver-side synthesis), the relay sees only a `SynthChatMessage` order — no voice traffic generated. The receiver's client synthesizes locally.

---

### Implementation Estimate

| Component                                                    | Crate(s)              | Est. Lines | Phase    |
| ------------------------------------------------------------ | --------------------- | ---------- | -------- |
| STT backend trait + local Whisper                            | `ic-audio`            | ~400       | Phase 5  |
| TTS backend trait + local Piper                              | `ic-audio`            | ~350       | Phase 5  |
| Caption overlay UI                                           | `ic-ui`               | ~250       | Phase 5  |
| Sender-side TTS pipeline                                     | `ic-audio`, `ic-net`  | ~200       | Phase 5  |
| TTS text preprocessing (rule-based)                          | `ic-audio`            | ~150       | Phase 5  |
| Extended mute model                                          | `ic-game`             | ~80        | Phase 5  |
| Console commands                                             | `ic-game`             | ~100       | Phase 5  |
| Settings UI (voice-text bridge panel)                        | `ic-ui`               | ~200       | Phase 5  |
| Cloud backend adapters (Azure, Google, ElevenLabs, Deepgram) | `ic-audio`            | ~300       | Phase 6a |
| Voice persona system + Workshop packs                        | `ic-audio`, `ic-game` | ~350       | Phase 6a |
| Faction-themed auto-assignment                               | `ic-game`             | ~100       | Phase 6a |
| TTS preprocessing (spell + statistical + WFST)               | `ic-audio`            | ~200       | Phase 6a |
| TTS preprocessing (LLM/SLM tier, optional)                   | `ic-audio`            | ~100       | Phase 7  |
| Receiver-side TTS pipeline (Mode B)                          | `ic-audio`, `ic-net`  | ~150       | Phase 5  |
| Cross-language translation pipeline                          | `ic-audio`            | ~250       | Phase 7  |
| **Total**                                                    |                       | **~3,180** |          |

### Alternatives Considered

| Alternative                                    | Verdict  | Reason                                                                                                                                                               |
| ---------------------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Platform-level only (rely on Xbox/OS STT/TTS)  | Rejected | Only works on Xbox/Windows. Linux, macOS, WASM, Steam Deck have no platform STT/TTS. IC must be cross-platform                                                       |
| Always-on captions (opt-out instead of opt-in) | Rejected | STT processing costs CPU. Default-off respects players who don't need it and avoids surprise resource usage                                                          |
| Single TTS voice for all players               | Rejected | Multiple players using the same voice is confusing — "who said that?" Per-player personas solve attribution                                                          |
| Cloud-only backends                            | Rejected | Privacy concern (voice data leaves device). Offline play broken. Local backends must be the default; cloud is optional upgrade                                       |
| Voice cloning of other players                 | Rejected | Consent and safety concern. Players can only clone/customize their *own* voice persona. Impersonation is explicitly blocked                                          |
| Real-time voice style transfer (voice changer) | Deferred | Different from TTS — this is modifying live voice, not synthesizing from text. Interesting but orthogonal to the voice-text bridge. Could be a future D059 extension |

### Cross-References

- **Communication system:** [D059](D059-communication.md) (VoIP architecture, text chat, muting model, relay forwarding, Opus codec)
- **Player profile:** [D053](../09e/D053-player-profile.md) (reference only — D079 explicitly does NOT store voice persona in D053 profile; uses session capability advertisement instead)
- **Community servers:** [D052](../09b/D052-community-servers.md) (relay forwarding, moderation)
- **LLM provider setup:** [`player-flow/llm-setup-guide.md`](../../player-flow/llm-setup-guide.md) (pluggable backend configuration pattern)
- **Workshop:** [D050](../09c/D050-workshop-library.md) (community voice persona packs)
- **Settings:** [`player-flow/settings.md`](../../player-flow/settings.md) (voice-text bridge settings panel)
- **Console commands:** [D058](D058-command-console.md) (`/captions`, `/tts`, `/voice-persona`, `/player <name> mute`)
