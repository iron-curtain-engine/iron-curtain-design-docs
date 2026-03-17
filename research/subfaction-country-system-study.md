# Subfaction & Country System — Research Study

> **Purpose:** Evaluate how RTS games implement subfaction/country differentiation systems, identify what creates fun balance, and recommend an approach for Iron Curtain that respects the alternate timeline (no Eastern Bloc) and aligns with D019 balance philosophy.
>
> **Date:** 2026-03-16
> **Status:** Research proposal (not yet adopted — requires D019 integration and community feedback)

---

## Industry Research

### How Existing Games Handle Subfactions

| Game | Approach | Country/Subfaction Count | What Worked | What Failed |
|---|---|---|---|---|
| **Red Alert 1** (1996) | Pure passive bonuses: +10% armor, firepower, fire rate, cost, or speed | 3 Allied (England, France, Germany) + 2 Soviet (Russia, Ukraine) = 5 | Simple, clear, low balance cost | Not interesting — doesn't change how you play. France/England bonuses were bugged (reversed) in original release |
| **Red Alert 2** (2000) | Country bonuses with unique units per nation | 5 Allied + 4 Soviet = 9 | Created memorable identities: Iraq's Desolator is iconic; America's free Paradrops provide consistent economic advantage | Sharp tier lists — Iraq dominated competitively, Russia considered "most useless Soviet side." Map-dependent viability (naval maps favor Allies, small maps favor Soviet rushes) |
| **Generals: Zero Hour** (2003) | Full subfactions — 3 generals per faction with deep unit/ability differentiation | 3 factions × 3 generals = 9 | High variety, distinct playstyles. Original 3 base factions were well-balanced | 9×9 matchup matrix was impossible to balance. Infantry General's Assault Troop Crawler "rather broken." Jump from 3 balanced factions to 12 unbalanced subfactions |
| **Age of Empires 2** (1999–present) | Shared tech tree with holes + 1-2 unique units + thematic passives per civ | 45+ civilizations | Scales to enormous civ counts; AoE2 pattern (inherit base, remove specific items, add 1-2 unique items) is the gold standard for subfaction design | Only ~5 civs viable per map at competitive level. 45+ options with minor differences can overwhelm new players |
| **StarCraft 2** (2010) | Zero subfactions, 3 pure factions | 3 | Most competitively balanced RTS ever made. Blizzard deliberately rejected a 4th race to focus design energy | Doesn't solve mirror-match staleness. IC has only 2 factions, making mirrors more common — subfactions specifically address this |
| **Company of Heroes** (2006) | Doctrine chosen pre-game, 3 per faction. Each doctrine has two upgrade paths with abilities costing Command Points | 4 factions × 3 doctrines = 12 | Creates strategic depth as a pre-game decision. Doctrine choice reflects map and opponent reading | Ongoing balance struggles. "Even games with longer development cycles like COH2 have struggled to make all commander choices attractive." |
| **C&C3: Kane's Wrath** (2008) | 9 total factions organized as variants within 3 faction "families" | 3 families × 3 variants = 9 | Identity with development efficiency. Family structure is intuitive | Tier lists still emerge despite minor differences |

### Key Analytical Findings

**From Wayward Strategy's "A Case for Sub-Factions" (2021):**

- Subfactions are the best middle ground between minimal asymmetry (AoE2) and total asymmetry (StarCraft). They provide "personalization" while maintaining reasonable balance overhead.
- **Mirror match solution:** Subfactions provide players with unique strategic paths even in mirror matches — critical for a 2-faction game like IC.
- **Content efficiency:** Meaningful variety without producing entirely new unit rosters, art assets, or tech trees.
- Passive bonuses (percentage-based) are easier to balance but less strategically meaningful. Unique units create distinct gameplay but require careful testing across all matchups. **Tech tree modifications offer the best fun-per-balance-cost ratio.**
- Even with minor bonuses, tier lists are inevitable. The only way to manage them is "a long cycle of analysis and balancing."
- Subfactions are easier to balance than loadout/deck systems and have superior **clarity** — players understand what they're getting.

**From RA1 Competitive Community (CnCNet):**

- Original RA1 had 5 countries with 10% bonuses and a famous bug (England/France reversed). Despite minimal differentiation, a tier list existed.
- Turkey, Greece, Spain were campaign-only in original RA1; added to multiplayer later with no bonuses.

**From RA2 Competitive Community (CnCNet/XWIS):**

- Iraq (Desolator) was consensus Tier 1 Soviet. Russia (Tesla Tank) was consensus bottom tier.
- America (free Paradrops) was consensus strongest Allied. The economic advantage was more impactful than combat bonuses.
- Map type dramatically shifted viability — naval maps favor Allies, small maps favor Soviet rushes.
- Unique units created sharper identity than passive bonuses but also sharper imbalance.

**From StarCraft 2 Design Philosophy:**

- Blizzard separated single-player and multiplayer balance completely — campaign had "spicier" units removed from competitive play.
- "Hyper-precise games such as StarCraft 2 probably wouldn't be the best place for a subfaction system."
- IC is not targeting SC2-level competitive precision — it targets the C&C community, which historically accepts moderate asymmetry.

---

## The Lore Constraint — No Eastern Bloc

In IC's alternate timeline, Einstein erases Hitler in 1924. Without WWII:

- **No division of Germany** — East/West Germany never existed. Germany is a unified Weimar-era democracy.
- **No Eastern Bloc** — Poland, Czechoslovakia, Hungary, Romania, Bulgaria are independent nations, never occupied by the Red Army as WWII spoils. There is no Warsaw Pact.
- **No Soviet satellite states** — the "Soviet countries" in real-world Cold War politics were products of WWII's outcome. They don't exist in IC's timeline.
- **Cuba has no reason to go communist** — the Cold War dynamics (Batista, Bay of Pigs, missile crisis) never occurred.

**Implication:** The Soviet faction IS the USSR — one monolithic state with constituent Soviet Socialist Republics (Russia, Ukraine SSR, Belarus SSR, Georgia SSR, etc.). These are not independent countries with distinct military traditions in the way Allied nations are. "Pick a country" doesn't map symmetrically.

---

## Recommended Design — Asymmetric Subfaction Identity

### The Core Asymmetry

**Allies pick a nation** — independent countries with distinct geography, military traditions, and national identity. You're defending your homeland. It's personal and patriotic.

**Soviets pick an institution** — competing power structures within the totalitarian state. You're a tool of the machine. It's cold, doctrinal, and ideological.

This reflects how each side actually works. The Alliance is a coalition of sovereign nations cooperating. The USSR is a monolithic state where parallel institutions compete for Stalin's favor.

**Mechanically, both sides work identically:** pick one from a list, receive one passive bonus + one tech tree modification. The asymmetry is in the narrative framing, not in the systems — meaning **reasonable balance overhead** (one passive + one tech mod per subfaction, validated through telemetry before ranked promotion — see § Ranked Viability below).

### Structure Per Subfaction (D019-Aligned)

Each subfaction gets:

1. **One thematic passive** — nudges toward a playstyle, not a raw stat bump. (D019 Principle 1: asymmetry creates identity)
2. **One tech tree modification** — access to one unit/upgrade earlier, one unique upgrade, or one standard unit replaced by a variant. (D019 Principle 5: balance through addition, not subtraction)
3. **No unique-from-scratch units at launch** — variants of existing units (different stats, visual distinction) yes; entirely new unit archetypes no. Unique units come later via Workshop when balance data exists. (D019 Principle 6: patch sparingly, observe patiently)

### Number: 4 Per Side

Research consistently shows diminishing returns past 4-5 options. 4 Allied × 4 Soviet = 16 cross-faction matchups, which is testable. 6×6 = 36 is not (at launch). More options are added via modding and Workshop.

### Allied Nations

Each country's identity reflects the alternate timeline's 22-year vacuum (1924–1946), not generic military tropes.

| Country | Passive | Tech Tree Mod | Playstyle Identity | Timeline Rationale |
|---|---|---|---|---|
| **England** | Naval units gain +1 sight range and +10% HP | Cruiser available at lower tech requirement | Naval control, information advantage | Island nation; 22 years of no continental threat amplified naval focus and reluctance to commit to land wars |
| **France** | Structures build 15% faster | Defensive structures gain a unique fortification upgrade | Turtling, established presence | Decades of peace bred institutional depth but weak offensive capability; the army is professional but small |
| **Germany** | Vehicle production 10% cheaper | Medium Tanks can be upgraded to an improved variant | Armor-heavy, industrial efficiency | Weimar democracy with unmatched industrial base; engineering expertise never rearmed but never disappeared |
| **Greece** | Infantry gain combat bonus on rough terrain | Access to Partisan squad (cheap militia variant of Rifle Infantry) | Scrappy, terrain-dependent, infantry-focused | The flashpoint nation; guerrilla tradition from internal instability; the most desperate and resourceful |

### Soviet Institutions

The USSR's internal power structure — competing organs that Stalin deliberately played against each other — maps directly to distinct playstyles and reinforces the totalitarian faction identity.

| Institution | Passive | Tech Tree Mod | Playstyle Identity | Lore Rationale |
|---|---|---|---|---|
| **Red Army** (Красная Армия) | Infantry and vehicle production 10% faster | Heavy Tank variant available (slower, more HP) | Overwhelming numbers, brute force | Deep Battle doctrine; mass and speed over precision; the conventional military arm |
| **NKVD** (State Security) | Enemy units near base defenses suffer -10% accuracy | Conscript unit available (very cheap, weak infantry — spam unit) | Area denial, expendable waves, oppression | Secret police and internal security; terror and control as military tools; penal battalions |
| **GRU** (Military Intelligence) | Spy actions 25% cheaper; radar range +15% | Access to a camouflaged scout vehicle | Information, infiltration, precision strikes | Military intelligence and Spetsnaz; the practical operators behind the ideological facade |
| **Soviet Science Bureau** | Tech center research 20% faster | Tesla Coil gains a unique overcharge upgrade | Tech rush, experimental power spikes | Racing to exploit Einstein's temporal research and Tesla technology; the experimental weapons arm |

### Balance Preset Integration (D019)

| Preset | Subfaction Behavior |
|---|---|
| **Classic** | RA1-original 10% passives (England +10% armor, France +10% fire rate, Germany +10% firepower, Russia -10% cost, Ukraine +10% speed). No unique units. Faithful recreation including only the original 5 countries. |
| **IC Default** | Expanded 4-per-side system with thematic passives + tech tree mods. The interesting system described above. |
| **Custom / Modded** | Community can add countries (Norway, Spain, Italy, Turkey), institutions, or entirely new subfaction categories via YAML inheritance (Tier 1 modding, D019 Principle 5 pattern). |

Classic preset players get the exact RA1 experience. IC Default players get the richer system. Nobody loses anything.

### Campaign Integration

In campaign, subfactions work dynamically rather than as a static pick:

- **Allied campaign:** The player doesn't pick a country — they ARE the Alliance. Operating in different theaters via the War Table grants **temporary theater bonuses** that echo country passives. A Greece operation yields partisans. A North Sea naval operation provides the UK naval bonus. This is more interesting than a static choice because it changes per operation.
- **Soviet campaign:** Different mission types carry institutional flavor. An NKVD suppression mission provides commissar bonuses and expendable conscripts. A GRU intelligence operation provides recon assets. A Red Army offensive provides mass production. The institution shapes the mission's available tools.
- Both campaigns use the **same YAML bonus definitions** as multiplayer. The campaign gets dynamic subfaction flavor for free through the War Table system.

### Ranked Viability

Start as casual/skirmish only. Promote to ranked after one full season of balance telemetry confirms:
- No subfaction has >55% win rate across all maps and rating brackets
- No subfaction has <10% pick rate (D019 Principle 7: low pick rate with high win rate means it's strong but unpleasant — redesign the bonus)
- Unit production diversity is not collapsed (if one subfaction only uses 3 of 15 units, the bonus is too narrow)

### Modding Contract (YAML)

Subfactions are Tier 1 YAML — no code required. Uses D019 Principle 5's inheritance pattern:

```yaml
# Example: Community-created Allied subfaction
subfaction:
  id: turkey_allied
  name: "Turkey"
  inherits: allied
  passive:
    description: "Border Fortress — walls and base defenses cost 15% less"
    modifiers:
      - target: { category: defense }
        stat: cost
        multiply: 0.85
  tech_tree_mod:
    description: "Fortified Bunker — garrisonable defensive structure replacing Pillbox"
    replacements:
      - original: pillbox
        replacement: fortified_bunker
  llm:
    identity: "Positional defense, map control, turtling"
    counters: ["aggressive early rushers"]
    countered_by: ["long-range siege, air power"]
```

---

## Relationship to Existing Design

- **D019 (Balance Presets):** Subfactions are a D019 extension. Principle 5 already documents the YAML inheritance pattern for exactly this. Balance preset integration is native.
- **D021 (Campaigns):** Campaign theater bonuses are an application of subfaction passives through the War Table, not a separate system.
- **D043 (AI Commanders):** AI commanders and subfactions are orthogonal. An AI can be "Col. Volkov — Armor" (commander personality) AND play as "Red Army" (subfaction bonus). The commander determines behavior; the subfaction determines bonuses.
- **D016 (LLM Factions):** LLM-generated factions are full custom factions. Subfactions are minor variations on existing factions. Different scope, same YAML infrastructure.

---

## Sources

- [Have it your way: a case for sub-factions — Wayward Strategy (2021)](https://waywardstrategy.com/2021/02/17/subfactions/)
- [Nation Tier List — RA2 CnCNet Forums](https://forums.cncnet.org/topic/4893-nation-tier-list/)
- [Countries Bonuses — RA1 CnCNet Forums](https://forums.cncnet.org/topic/3845-countries-bonuses/)
- [C&C Generals Zero Hour Balance — SpaceBattles](https://forums.spacebattles.com/threads/command-conquer-generals-original-zh-balance.295366/)
- [Does StarCraft 2's Strategy Design Still Succeed? — Game Wisdom](https://game-wisdom.com/critical/starcraft-2)
- [How to balance asymmetric civilizations — AoE Forum](https://forums.ageofempires.com/t/how-to-balance-asymmetric-civilizations/86214)
- [Best Allied Nation? — Remastered Collection Steam Discussion](https://steamcommunity.com/app/1213210/discussions/0/3105763714520557556/)
