## D005: Modding — WASM for Power Users (Tier 3)

**Decision:** WASM modules via `wasmtime`/`wasmer` for advanced mods.

**Rationale:**
- Near-native performance
- Perfectly sandboxed by design
- Deterministic execution (critical for multiplayer)
- Modders can write in Rust, C, Go, AssemblyScript, or Python-to-WASM
- Leapfrogs OpenRA (requires C# for deep mods)

**Full specification:** [`modding/wasm-modules.md`](../modding/wasm-modules.md) — includes WASM Host API, capability-based security model, install-time permission prompts (mobile-app pattern), execution resource limits, float determinism, rendering/pathfinding/AI/format-loader API surfaces, mod testing framework, and deterministic conformance suites.
