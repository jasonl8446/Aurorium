# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Aurorium is a Rust service (edition 2024) for the Revive101 project. It polls the Wizard101 patch server for new game revisions, downloads the changed assets, tracks them in SQLite, and serves them over HTTP.

## Commands

```bash
cargo run                 # debug run (generates config.toml on first launch)
cargo build --release     # optimized build (lto, codegen-units=1, panic=abort, stripped)
cargo clippy --all-targets
cargo fmt
```

There is no test suite and no CI configuration in the repo. To exercise behaviour, run the binary and hit the HTTP routes.

`config.toml`, `data/`, `logs/`, and `aurorium.db*` are gitignored — they are all runtime artifacts created on first run.

## Architecture

`main.rs` loads `AppConfig`, initializes `tracing` and the `Database`, then runs two futures in a `tokio::select!`:

1. **`revision_checker`** — an infinite loop that sleeps `fetcher.fetch_interval` seconds between passes. Each pass: `WizardPatcher::check_revision` → `ManifestFetcher` → `Database::insert_new_revision` → `AssetFetcher::fetch_assets`. Failures inside a pass are logged and retried on the next tick; the loop must never return early or the whole process exits via `select!`.
2. **`file_server`** — axum router (`/revisions`, `/latest`, `/{revision}/{*file_path}`) sharing `AppState { config, db }`, served with `into_make_service_with_connect_info::<SocketAddr>()` (required by the `ConnectionAddr` extractor).

### Patch-server handshake (`wizard_patcher.rs`)

Raw TCP against `patch.us.wizard101.com:12500`. Reads a 28-byte session offer, writes the hardcoded `SESSION_ACCEPT` hex blob (PatchMessages(8) → MSG_LATEST_FILE_LIST_V2(2)), then parses the reply out of a fixed 256-byte buffer: a `FOOD` header (`0x0D 0xF0`), skipped byte runs, and length-prefixed bytestrings yielding `list_file_url` and `url_prefix`. Byte offsets and buffer sizes here are protocol constants — changing them silently breaks parsing or overruns the buffer. All socket ops are wrapped in a 30s `TCP_TIMEOUT`.

The revision (`V_r773351.Wizard_1_570_0_Live`) is regex-extracted from the list-file URL; the numeric part is what orders revisions in the DB.

### Manifests and assets

`ManifestFetcher` downloads `LatestFileList.bin` and `LatestFileList.xml` into `{save_directory}/{revision}/`, skipping if already present. `xml_parser::parse_file_list` is a hand-rolled streaming `quick-xml` reader (not serde) that emits `Asset` records from `<RECORD>` elements, ignoring anything nested under `_TableList`/`About`.

`AssetFetcher` downloads via `stream::buffer_unordered(concurrent_downloads)` with an `indicatif` `MultiProgress`. Downloads that fail are logged and skipped, not retried — the next revision-check pass picks them up because on-disk existence is the skip check.

The shared `Fetcher` trait (`fetcher/fetcher.rs`) provides `write_to_file_streamed`, which always writes to `path.part` first and renames on success, deleting the partial on error. Keep that pattern for any new download path.

### Asset deduplication (`db.rs`)

Two tables created by a single `rusqlite_migration` migration. `assets` is keyed `(revision, file_name)` and carries an `origin_revision` column — the revision that *first* introduced that exact `(file_name, crc, size)` triple. `insert_new_revision` looks each asset up by that triple; if a prior origin exists the row points back at it and the file is **not** re-downloaded, so bytes live on disk only under their origin revision's directory.

`routes/file.rs` mirrors this: it resolves the requested path through `get_revision_for_asset` to find the owning revision before serving. `LatestFileList*` is the exception — always served from the requested revision directly. The route also rejects `..`/absolute components before touching the filesystem.

Adding a migration means appending a new `M::up(...)` to the `MIGRATIONS` vec — never edit the existing entry, deployed databases have already applied it.

### Errors and logging

All error enums live in one file, `errors.rs`, as `thiserror` + `miette::Diagnostic` types with a `code(...)` and a user-facing `help(...)`. New fallible code should add a variant there rather than returning bare `miette::miette!` strings. `RouteError`'s `IntoResponse` impl lives in `routes/file.rs`.

`init_logging` returns a `WorkerGuard` that must stay alive for file logging to flush. The console filter is scoped to the crate (`error,aurorium={level}`) and is overridden entirely by `RUST_LOG` if set.

## Notes

- `Cargo.toml` carries a `# THIS MUST STAY ON 0.42.0` comment above `quick-xml` (currently pinned to `0.41.0`) — confirm with the maintainer before bumping it; the parser depends on its event API.
- `README.md` documents every `config.toml` field; update that table when adding a config option.
