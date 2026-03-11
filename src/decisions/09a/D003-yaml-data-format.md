## D003: Data Format — Real YAML, Not MiniYAML

**Decision:** Use standard spec-compliant YAML with `serde_yaml`. Not OpenRA's MiniYAML.

**Rationale:**
- Standard YAML parsers, linters, formatters, editor support all work
- `serde_yaml` → typed Rust struct deserialization for free
- JSON-schema validation catches errors before game loads
- No custom parser to maintain
- Inheritance resolved at load time as a processing pass, not a parser feature

**Alternatives considered:**
- MiniYAML as-is (rejected — custom parser, no tooling support, not spec-compliant)
- TOML (rejected — awkward for deeply nested game data)
- RON (rejected — modders won't know it, thin editor support)
- JSON (rejected — too verbose, no comments)

**Migration:** `miniyaml2yaml` converter tool in `cnc-formats` crate (behind `miniyaml` feature flag, MIT/Apache-2.0). `ra-formats` wraps this for IC's runtime auto-conversion pipeline (D025).
