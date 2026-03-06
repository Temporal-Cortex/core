# Temporal Cortex Core

Public library: TOON serialization format + Truth Engine for deterministic temporal reasoning. MIT/Apache-2.0 dual-licensed.

## Crate Structure

| Crate | Description |
|-------|-------------|
| `toon-core` | TOON encoder/decoder (Rust) |
| `toon-cli` | CLI tool for TOON ↔ JSON conversion |
| `toon-wasm` | WebAssembly bindings for TOON |
| `toon-python` | Python bindings (PyO3) |
| `truth-engine` | RRULE expansion, conflict detection, availability computation |
| `truth-engine-wasm` | WebAssembly bindings for Truth Engine |

JS/TS packages: `packages/toon-js`, `packages/truth-engine-js`

## Build & Test

```
cargo test --workspace
cargo fmt --all
cargo clippy --workspace --all-targets -- -D warnings
```

WASM builds (requires wasm-bindgen-cli matching Cargo.lock version):

```
pnpm build:wasm
```

Python bindings: `cd toon-python && maturin develop && pytest`

## Conventions

- CI rejects formatting diffs (`cargo fmt`) and clippy warnings (`-D warnings`)
- 510+ Rust, 42 JS, 30 Python, ~9,000 property-based tests
- No pricing or business strategy content (public repo)
- TOON format rules: no exponents, no leading zeros, no trailing fractional zeros in numbers
- Strings: quote if empty, looks-like-bool/null/number, or contains colon/backslash/brackets/control chars
- Keys: unquoted if `^[A-Za-z_][A-Za-z0-9_.]*$`

## Boundaries

- Always: run `cargo fmt --all` and `cargo clippy` before committing
- Ask first: changing public API signatures or TOON format rules
- Never: include pricing, business strategy, or proprietary Platform code
