# PHASE 1 — Core Shell: Detailed Plan

**Parent:** [`MAIN_PLAN.md`](./MAIN_PLAN.md) → Phase 1 (PRD §13 M1)
**Goal:** SQLite schema + migrations, XDG-correct storage, settings persistence, typed command layer, three-region layout skeleton.
**PRD references:** §4 (architecture), §5.1 (schema), §5.4 (storage), §2 theming row, §11 (pragmas/perf groundwork).
**Prerequisite:** Phase 0 complete (git baseline, Tailwind wired, plugins registered, CSP set, lint/test tooling in place).
**Estimated scope:** 2–3 working sessions. First phase with real Rust feature code.

---

## 1. Starting State (after Phase 0)

- Three-region Tailwind shell with no logic; `dialog` + `clipboard-manager` plugins registered; CSP enforced; Vitest/ESLint/clippy pipeline green.
- Rust side: `rusqlite 0.40.2` (feature `bundled`) and `tokio` (full) already in `Cargo.toml`, **unused**. No `directories` crate yet. No state, no commands beyond the Phase 0 temporary `ping`.
- Identifier: `com.adev.offline-docs` — matters for path decisions below.

---

## 2. Decisions to Lock In

1. **XDG paths: use the `directories` crate, not Tauri's `app_data_dir()`.** Tauri returns `$XDG_DATA_HOME/<identifier>` = `$XDG_DATA_HOME/com.adev.offline-docs/`, but the PRD §5.4 pins `$XDG_DATA_HOME/offline-docs/`. `directories::ProjectDirs::from("", "", "offline-docs")` gives `data_dir`, `config_dir`, `cache_dir` with the exact XDG semantics and fallbacks the PRD wants. Add `directories = "6"` to `Cargo.toml`.
2. **Concurrency model: async Tauri commands + `spawn_blocking`.** Sync commands run on Tauri's main thread and would freeze the UI during queries; and a `std::sync::Mutex<Connection>` guard must never be held across an `.await`. Pattern for every DB command:
   ```rust
   let state = state.inner().clone(); // Arc<AppState>
   tauri::async_runtime::spawn_blocking(move || { let _g = state.db.lock()...; ... })
       .await.map_err(|e| AppError::from(e))?
   ```
   `AppState = { db: Arc<Mutex<Connection>>, paths: Paths }` managed via `.manage()`. (No connection pool — one writer connection is correct for SQLite here; r2d2 is YAGNI.)
3. **Migrations: plain SQL embedded via `include_str!`, tracked by PRAGMA `user_version`.** No diesel/refinery — three migrations total are foreseeable in V1; a hand-rolled runner in `db/migrations.rs` is the KISS choice. Migration files: `src-tauri/src/db/migrations/001_core_schema.sql`, `002_fts.sql` (added in Phase 4).
4. **FTS5 availability (MAIN_PLAN risk #1):** verify, don't assume. The `bundled` feature of `libsqlite3-sys` historically enables FTS5, but this must be **asserted at startup** (`SELECT fts5('x')` / create a temp virtual table). If it fails, the fix is switching the feature to `bundled-full` (or adding `"functions"`); keep that one-line change ready. The startup assert ships in Phase 1 so Phase 4 never discovers this late.
5. **Error type: single `AppError`** (`thiserror`) with variants `Db(rusqlite::Error)`, `Io(std::io::Error)`, `Serialization(serde_json::Error)`, `Message(String)`, converted to `String` at the command boundary (`impl Serialize`). Frontend gets one predictable error shape: `{ kind, message }`.
6. **`bookmarks` table is created now, feature-less** — per PRD §5.1/§10, so later migrations don't need to backfill it.

---

## 3. Task Breakdown (ordered)

### Task 1 — Paths module (§5.4)
- `src-tauri/src/paths.rs`: `struct Paths { data_dir, config_dir, cache_dir }`, built from `ProjectDirs`, all three `.create_dir_all()`-ed at startup.
- Resolved locations:
  | Data | Path |
  |---|---|
  | DB | `<data_dir>/library.sqlite3` |
  | Settings | `<config_dir>/settings.json` |
  | Staging (Phase 2) | `<cache_dir>/import/` (created lazily) |
- Unit-testable: accept a base-dir override for tests (the resolver takes an optional `dirs` supplier) so tests never touch the real `$HOME`.
- **XDG verification step (explicit):** run once with `XDG_DATA_HOME=/tmp/xdg-test XDG_CONFIG_HOME=... XDG_CACHE_HOME=... pnpm tauri dev` and confirm the files land under the override — this discharges MAIN_PLAN risk #4 with evidence.

### Task 2 — DB open + pragmas + migrations (§5.1, §11)
- `src-tauri/src/db/mod.rs`: open `library.sqlite3` (`Connection::open`), then set:
  - `PRAGMA journal_mode = WAL;` (better read concurrency; page-load latency goal §11)
  - `PRAGMA foreign_keys = ON;` — **required**: schema relies on `ON DELETE CASCADE` for versions/pages/bookmarks
  - `PRAGMA synchronous = NORMAL;` (WAL-appropriate durability/speed tradeoff)
- `src-tauri/src/db/migrations.rs`: runner reads `PRAGMA user_version`, applies each embedded migration ≥ current version inside a transaction, bumps `user_version`. Idempotent by construction; a partially-applied migration rolls back cleanly.
- **Migration 001** = the §5.1 DDL verbatim:
  - `docs` (name UNIQUE, slug UNIQUE, description, icon, created_at)
  - `versions` (doc_id FK CASCADE, version, is_default, imported_at, UNIQUE(doc_id, version))
  - `pages` (version_id FK CASCADE, parent_id FK CASCADE, title, slug, content, raw_source, sort_order, UNIQUE(version_id, slug))
  - `bookmarks` (page_id FK CASCADE, note, created_at)
  - indices: `idx_versions_doc(doc_id)`, `idx_pages_version(version_id)`, `idx_pages_parent(parent_id)`
- Rust unit tests: run migrations on an in-memory DB twice (idempotency); assert all tables + indices exist via `sqlite_master`; assert `ON DELETE CASCADE` actually cascades (insert doc→version→page, delete doc, assert page gone).

### Task 3 — FTS5 startup assert
- In `db/mod.rs` after migrations: `CREATE VIRTUAL TABLE temp.fts5_probe USING fts5(x);` + drop it; fail startup with a clear message if unavailable (Decision 4). Marked `#[cfg(test)]`-covered too.

### Task 4 — Settings (§2 theming, §5.4)
- `src-tauri/src/settings.rs`:
  ```rust
  #[derive(Serialize, Deserialize)]
  #[serde(default)]
  struct Settings { theme: Theme, last_opened_doc: Option<String> } // Theme = system|light|dark, default "system"
  ```
- Load: missing/corrupt file → defaults (never crash on a bad settings.json; log + recreate).
- Save: **atomic write** — write `settings.json.tmp`, then `rename` (a crash mid-write must not corrupt settings).
- Not in SQLite (PRD §5.4 puts it in the config dir explicitly).

### Task 5 — Command layer (§4)
- `src-tauri/src/commands/` with one file per domain; all async + `spawn_blocking` per Decision 2:
  - `list_docs() -> Vec<DocDto>` — **no content columns**, cheap metadata only (§11 no-`SELECT *` rule starts here)
  - `list_versions(doc_id) -> Vec<VersionDto>`
  - `get_page(page_id) -> PageDto` — includes `content` + `raw_source`; the only command that fetches content
  - `get_settings() / set_settings(settings)`
  - keep Phase 0's temporary `ping` removed once the first real command is wired
- DTOs in `src-tauri/src/models.rs` (serde `rename_all = "camelCase"` so the frontend TS types match naturally).
- TS side (`src/lib/api.ts`): typed wrappers `listDocs()`, `listVersions()`, `getPage()`, plus hand-written interfaces mirroring the DTOs; single `invokeErrorToString()` helper. This file becomes the **only** place the frontend knows about Tauri (§4 trust boundary, keeps React testable).

### Task 6 — Frontend layout skeleton (§8)
- `src/App.tsx` → three real regions as components (no logic yet):
  - `TopBar` — search input (disabled placeholder "Search all documentation" until Phase 4) + settings gear icon
  - `Sidebar` — "Library" heading, empty-state text ("No documentation imported yet"), Import button placeholder (non-functional until Phase 2)
  - `ContentPane` — empty-state ("Select a page")
- `src/hooks/useSettings.ts` — loads settings on mount, exposes `theme` + setter; theme value applied to `document.documentElement` (`classList.toggle('dark')` / `data-theme`) so Tailwind dark-mode works. **Actual GTK `prefer-dark` following is wired in Phase 3** (needs Tauri's theme API + window config); Phase 1 only persists the setting and applies the class.
- Vitest: render the three regions; `useSettings` defaults to `system`.

### Task 7 — Wiring in `lib.rs`
- `run()` becomes: resolve paths → open DB + migrate + FTS5 assert → load settings → `.manage(AppState { … })` → `.invoke_handler(generate_handler![…])`. Startup failures (DB unopenable, FTS5 missing) exit with a clear stderr message, not a panic-void.
- `cargo clippy -- -D warnings` + `cargo fmt` clean; `tsc --noEmit` clean.

---

## 4. Verification Checklist (acceptance — extends MAIN_PLAN Phase 1)

Automated:
- [ ] `cargo test`: migration idempotency, cascade-deletes, settings round-trip (incl. corrupt-file recovery), path-resolver unit tests.
- [ ] `pnpm test`: layout renders; settings hook defaults.

Manual (the MAIN_PLAN acceptance items, made concrete):
- [ ] Fresh launch creates `$XDG_DATA_HOME/offline-docs/library.sqlite3` (or `~/.local/share/offline-docs/` when unset) containing all 4 tables + 3 indices; `settings.json` appears in the config dir.
- [ ] Delete the data dir → relaunch recreates everything (migration idempotency in real conditions).
- [ ] With XDG overrides set, files land in the overridden dirs.
- [ ] `get_page`/`list_docs` round-trip through a dev-only debug panel (temporary component, removed before Phase 2) — or via devtools `invoke` calls.
- [ ] Theme value set via devtools call persists in `settings.json` across restart.
- [ ] UI never freezes during a query (async command pattern working — quick check with a deliberately slow test query).

## 5. Handoff to Phase 2

Phase 1 leaves behind: `AppState` with a migrated DB + paths + settings, the `AppError`/DTO/api.ts conventions every later phase reuses, and a content-free `pages` table ready for the importer. **Nothing here parses markdown, reads bundles, or creates the FTS table** — those are Phases 2 and 4. The Import button remains a visible placeholder (users see where it will live; it does nothing yet).
