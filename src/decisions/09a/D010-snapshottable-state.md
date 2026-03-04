## D010: Simulation — Snapshottable State

**Decision:** Full sim state must be serializable/deserializable at any tick.

**Rationale enables:**
- Save games (trivially)
- Replay system (initial state + orders)
- Desync debugging (diff snapshots between clients at divergence point)
- Rollback netcode (restore state N frames back, replay with corrected inputs)
- Cross-engine reconciliation (restore from authoritative checkpoint)
- Automated testing (load known state, apply inputs, verify result)

**Crash-safe serialization (from Valve Fossilize):** Save files use an append-only write strategy with a final header update — the same pattern Valve uses in Fossilize (their pipeline cache serialization library, see `research/valve-github-analysis.md` § Part 3). The payload is written first into a temporary file; only after the full payload is fsynced does the header (containing checksum + payload length) get written atomically. If the process crashes mid-write, the incomplete temporary file is detected and discarded on next load — the previous valid save remains intact. This eliminates the "corrupted save file" failure mode that plagues games with naïve serialization.

**Autosave threading:** Autosave (including `delta_snapshot()` serialization + LZ4 compression + fsync) MUST run on the dedicated I/O thread — never on the game loop thread. On a 5400 RPM HDD, the `fsync()` call alone takes 50–200 ms (waits for platters to physically commit). Even though delta saves are only ~30 KB, fsync latency dominates. The game thread's only responsibility is to produce the `DeltaSnapshot` data (reading ECS state — fast, ~0.5–1 ms for 500 units via `ChangeMask` bitfield iteration). The serialized bytes are then sent to the I/O thread via the same ring buffer used for SQLite events. The I/O thread handles file I/O + fsync asynchronously. This prevents the guaranteed 50–200 ms HDD hitch that would otherwise occur every autosave interval.

**Delta encoding for snapshots:** Periodic full snapshots (for save games, desync debugging) are complemented by **delta snapshots** that encode only changed state since the last full snapshot. Delta encoding uses property-level diffing: each ECS component that changed since the last snapshot is serialized; unchanged components are omitted. For a 500-unit game where ~10% of components change per tick, a delta snapshot is ~10x smaller than a full snapshot. This reduces save file size, speeds up autosave, and makes periodic snapshot transmission (for late-join reconnection) bandwidth-efficient. Inspired by Source Engine's `CNetworkVar` per-field change detection (see `research/valve-github-analysis.md` § 2.2) and the `SPROP_CHANGES_OFTEN` priority flag — components that change every tick (position, health) are checked first during delta computation, improving cache locality. See `10-PERFORMANCE.md` for the performance impact and `09-DECISIONS.md` § D054 for the `SnapshotCodec` version dispatch.
