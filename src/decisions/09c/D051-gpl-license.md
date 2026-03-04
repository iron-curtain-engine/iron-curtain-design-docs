## D051: Engine License — GPL v3 with Explicit Modding Exception

**Decision:** The Iron Curtain engine is licensed under **GNU General Public License v3.0** (GPL v3) with an explicit **modding exception** that clarifies mods loaded through the engine's data and scripting interfaces are NOT derivative works.

**Rationale:**

1. **The C&C open-source community is a GPL community.** EA released every C&C source code drop under GPL v3 — Red Alert, Tiberian Dawn, Generals/Zero Hour, and the Remastered Collection engine. OpenRA uses GPL v3. Stratagus uses GPL-2.0. Spring Engine uses GPL-2.0. The community this project is built for lives in GPL-land. GPL v3 is the license they know, trust, and expect.

2. **Legal compatibility with EA source.** `ra-formats` directly references EA's GPL v3 source code for struct definitions, compression algorithms, and lookup tables (see `05-FORMATS.md` § Binary Format Codec Reference). GPL v3 for the engine is the cleanest legal path — no license compatibility analysis required.

3. **The engine stays open — forever.** GPL guarantees that no one can fork the engine, close-source it, and compete with the community's own project. For a community that has watched proprietary decisions kill or fragment C&C projects over three decades, this guarantee matters. MIT/Apache would allow exactly the kind of proprietary fork the community fears.

4. **Contributor alignment.** DCO + GPL v3 is the combination used by the Linux kernel — the most successful community-developed project in history. OpenRA contributors moving to IC (or contributing to both) face zero license friction.

5. **Modders are NOT restricted.** This is the key concern the old tension analysis raised, and the answer is clear: YAML data files, Lua scripts, and WASM modules loaded through a sandboxed runtime interface are NOT derivative works under GPL. This is the same settled legal interpretation as:
   - Linux kernel (GPL) + userspace programs (any license)
   - Blender (GPL) + Python scripts (any license)
   - WordPress (GPL) + themes and plugins loaded via defined APIs
   - GCC (GPL) + programs compiled by GCC (any license, via runtime library exception)
   
   IC's tiered modding architecture (D003/D004/D005) was specifically designed so that mods operate through data interfaces and sandboxed runtimes, never linking against engine code. The modding exception makes this explicit.

6. **Commercial use is allowed.** GPL v3 permits selling copies, hosting commercial servers, running tournaments with prize pools, and charging for relay hosting. It requires sharing source modifications — which is exactly what this community wants.

**The modding exception (added to LICENSE header):**

```
Additional permission under GNU GPL version 3 section 7:

If you modify this Program or any covered work, by linking or combining
it with content loaded through the engine's data interfaces (YAML rule
files, Lua scripts, WASM modules, resource packs, Workshop packages, or
any content loaded through the modding tiers described in the
documentation as "Tier 1", "Tier 2", or "Tier 3"), the content loaded
through those interfaces is NOT considered part of the covered work and
is NOT subject to the terms of this License. Authors of such content may
choose any license they wish.

This exception does not affect the copyleft requirement for modifications
to the engine source code itself.
```

This exception uses GPL v3 § 7's "additional permissions" mechanism — the same mechanism GCC uses for its runtime library exception. It is legally sound and well-precedented.

**Alternatives considered:**

- **MIT / Apache 2.0** (rejected — allows proprietary forks that fragment the community; creates legal ambiguity when referencing GPL'd EA source code; the Bevy ecosystem uses MIT/Apache but Bevy is a general-purpose framework, not a community-specific game engine)
- **LGPL** (rejected — complex, poorly understood by non-lawyers, and unnecessary given the explicit modding exception under GPL v3 § 7)
- **Dual license (GPL + commercial)** (rejected — adds complexity with no clear benefit; GPL v3 already permits commercial use)
- **GPL v3 without modding exception** (rejected — would leave legal ambiguity about WASM mods that might be interpreted as derivative works; the explicit exception removes all doubt)

**What this means in practice:**

| Activity                                    | Allowed? | Requirement                                              |
| ------------------------------------------- | -------- | -------------------------------------------------------- |
| Play the game                               | Yes      | —                                                        |
| Create YAML/Lua/WASM mods                   | Yes      | Any license you want (modding exception)                 |
| Publish mods on Workshop                    | Yes      | Author chooses license (D030 requires SPDX declaration)  |
| Sell a total conversion mod                 | Yes      | Mod's license is the author's choice                     |
| Fork the engine                             | Yes      | Your fork must also be GPL v3                            |
| Run a commercial server                     | Yes      | If you modify the server code, share those modifications |
| Use IC code in a proprietary game           | No       | Engine modifications must be GPL v3                      |
| Embed IC engine in a closed-source launcher | Yes      | The engine remains GPL v3; the launcher is separate      |

### Phase

Resolved. The LICENSE file ships with the GPL v3 text plus the modding exception header from Phase 0 onward.

### CI Enforcement: cargo-deny for License Compliance

Embark Studios' **cargo-deny** (2,204★, MIT/Apache-2.0) automates license compatibility checking across the entire dependency tree. IC should add `cargo-deny` to CI from Phase 0 with a GPL v3 compatibility allowlist — every `cargo deny check licenses` run verifies that no dependency introduces a license incompatible with GPL v3 (e.g., SSPL, proprietary, GPL-2.0-only without "or later"). For Workshop content (D030), the `spdx` crate (also from Embark, 140★) parses SPDX license expressions from resource manifests, enabling automated compatibility checks at publish time. See `research/embark-studios-rust-gamedev-analysis.md` § cargo-deny.
