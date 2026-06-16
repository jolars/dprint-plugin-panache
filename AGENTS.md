# Agent instructions

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## What this is

A thin [dprint](https://dprint.dev) Wasm plugin that wraps the
[`panache-formatter`](https://crates.io/crates/panache-formatter) crate so the
Panache formatter (Quarto, Pandoc, R Markdown, Markdown) can run inside dprint.
The plugin holds no formatting logic of its own; it only maps dprint
configuration into a `panache_formatter::Config` and forwards the file text.

This crate is released independently of the main Panache CLI (which lives in the
`jolars/panache` repo). The separate repo exists so the `panache.wasm` release
asset does not pollute the Panache CLI's `v*` GitHub release stream, which other
tools (e.g. the Zed extension) resolve via `require_assets`.

## Build, lint, test

The crate only compiles for `wasm32-unknown-unknown` --- it relies on
`dprint_core::generate_plugin_code!`, so a native `cargo build` will not produce
a usable artifact. The wasm target is pinned in `rust-toolchain.toml`.

```bash
cargo build --release --target wasm32-unknown-unknown   # produces target/wasm32-unknown-unknown/release/dprint_plugin_panache.wasm
cargo fmt                                                 # rustfmt is a git-hook
cargo clippy --all-features --target wasm32-unknown-unknown
```

There are no in-crate unit tests. Correctness is enforced in CI
(`.github/workflows/ci.yml`) by a **parity + idempotency smoke test**: it builds
the wasm plugin, downloads the latest Panache CLI release, formats the same
sample through both, and `diff`s the outputs (they must match), then re-runs
`dprint fmt` to confirm the output is stable. When changing config mapping,
mirror this locally --- run a sample through `dprint fmt` with the built wasm
and compare against `panache format`. The plugin must stay byte-for-byte
identical to the CLI for equivalent settings.

Tooling (`dprint`, `cargo-audit`, `cargo-deny`, `cargo-llvm-cov`,
`cargo-flamegraph`, `wasm-pack`) comes from the devenv shell, not the global
profile, so it is only on PATH once the environment is loaded.

## Architecture

Everything lives in `src/lib.rs`:

- `Configuration` --- the dprint-facing config struct (serialized camelCase).
  All enum-valued options are stored as `String` here and parsed lazily.
- `parse_*` functions --- each maps a string config value to a
  `panache_formatter` enum, pushing a `ConfigurationDiagnostic` on an unknown
  value and falling back to a default. These run twice: once in `resolve_config`
  purely to collect diagnostics (results discarded into a throwaway vec), and
  again in `build_panache_config` to produce the real `Config`.
- `detect_flavor_from_path` --- `.qmd` → Quarto, `.rmd`/`.rmarkdown` →
  RMarkdown. **Path-based flavor detection takes precedence over the configured
  `flavor`** (see `build_panache_config`); only when the extension is
  inconclusive does the `flavor` config value apply.
- `build_panache_config` --- derives `parser_extensions` and
  `formatter_extensions` from the resolved flavor via `*::for_flavor(flavor)`,
  so flavor changes propagate to extensions automatically.
- `SyncPluginHandler` impl --- `resolve_config` reads dprint globals
  (`lineWidth` defaults to global `line_width` or 80; `tabWidth` to global
  `indent_width` or 4) and validates; `format` decodes UTF-8, calls
  `panache_formatter::format` inside `catch_unwind` (a formatter panic becomes
  an `anyhow` error rather than aborting the wasm instance), and returns `None`
  when output equals input.
- `generate_plugin_code!` --- generates the wasm plugin entrypoints; this is why
  the native build target is not useful.

`FILE_EXTENSIONS` is the list of extensions the plugin claims in dprint.

## Releasing

Versioning is managed by [versionary](https://github.com/jolars/versionary)
(`versionary.jsonc`, `release-type: rust`). Pushing a `v*` tag triggers
`publish-dprint-wasm.yml`, which builds the wasm, renames it to `panache.wasm`,
writes a `.sha256`, and uploads both to the matching GitHub release. The version
reported by the plugin and its `config_schema_url`/`update_url`
(`plugins.dprint.dev/jolars/panache/...`) come from `CARGO_PKG_VERSION`, so the
crate version in `Cargo.toml` must match the release tag.

When bumping the `panache-formatter` dependency, expect the `parse_*` functions
and `Config` field set to need updates if the upstream config API changed; the
CI build step exists specifically to catch this drift.
