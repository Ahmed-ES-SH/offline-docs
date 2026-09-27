# MAIN_PLAN — Offline Docs App Build Plan

**Source of truth:** [`PRD-offline-docs-app.md`](../PRD-offline-docs-app.md) (Draft v1, MVP scope)
**Plan created:** 2026-09-05
**Status of repo at planning time:** pristine `create-tauri-app` scaffold (Tauri 2.11 + React 19 + Vite 7 + TypeScript) with dependencies installed but **zero features built** — rusqlite declared but unused, Tailwind/shiki/react-router installed but not wired, clipboard/dialog/fs plugins declared in `Cargo.toml` but unregistered, capabilities minimal (`core:default` + `opener:default` only), CSP `null`, no git repo, no tests, no CI.

---

## 0. PRD Traceability Matrix

Every PRD section maps to at least one phase. Nothing in the PRD is dropped; §10 items are preserved as out-of-scope.

| PRD Section | Topic | Covered in |
|---|---|---|
| §1 | Problem & goal (offline-first docs reader) | Whole plan |
| §2 | Platform details (webkit2gtk-4.1, X11+Wayland, theming, desktop integration, XDG) | Phase 0, 3, 6 |
| §2.1 | AUR packaging (`offline-docs` + `offline-docs-bin`), AppImage | Phase 6 |
| §3 | Tech stack (Tauri 2, React+TS+Vite, rusqlite, FTS5, Tailwind, shiki) | Phase 0, 1 |
| §4 | Architecture (all SQLite in Rust, typed `invoke` commands) | Phase 1 |
| §5.1 | Core schema (docs/versions/pages/bookmarks + indices) | Phase 1 |
| §5.2 | FTS5 external-content table + sync triggers | Phase 4 |
| §5.3 | Query patterns (global + scoped, bm25, snippet, grouping) | Phase 4 |
| §5.4 | Storage locations (XDG data/config/cache) | Phase 1 |
| §6.1 | Content source priority (GitHub MD → `.md` endpoint/llms.txt → scraping) | Phase 5 |
| §6.2 | Decision: TypeScript CLI (Playwright + Turndown), no scraping framework | Phase 5 |
| §6.3 | Scraper architecture (`SourceConfig`, per-source configs, `cleanHtml`) | Phase 5 |
| §6.4 | Two-layer sanitization (scrape time + import time) | Phase 2, 5 |
| §6.5 | Per-page failure handling, no auto-retry | Phase 5 |
| §6.6 | Bundle format + Import action (folder/zip), transactional import | Phase 2 |
| §7.1/7.2 | Compile MD/MDX → sanitized HTML at import time; never eval MDX | Phase 2 |
| §7.3 | Tauri capability/permission lockdown | Phase 0, 1 |
| §7.4 | Strict CSP in `tauri.conf.json` | Phase 0 |
| §7.5 | Import validation (file types, size cap, zip-bomb, path traversal) | Phase 2 |
| §8 | UI/UX (sidebar tree, global search bar, version switcher, dark mode, copy button) | Phase 3 |
| §9 | Acceptance criteria (9 features) | Acceptance sections of each phase |
| §10 | Out of scope for V1 | Preserved (bookmarks table only, §Phase 1) |
| §11 | Non-functional (<100ms page, binary size, transactions, portability) | Phase 3, 4, 6 |
| §12 | Open questions | Resolved below |
| §13 | Milestones M1–M5 | Phases 1–4, 6 (Phase 0 and 5 added) |

---

## 1. Resolved Open Questions (PRD §12)

These decisions are baked into the plan; each is revisitable with a defined trigger.

1. **Syntax highlighting: `shiki`** (already in `package.json`). Grammars/themes bundled at build time so highlighting works with zero network. **Spike in Phase 3:** measure the size impact of bundled grammars on the final artifact. Fallback trigger: if bundled grammars push installed size past the 20–30 MB budget (§11), switch to `react-syntax-highlighter` (prism) with only the needed languages registered.
2. **MDX in V1: Markdown + a curated allow-list of safe components.** Headings, lists, tables, links, images, code fences, blockquotes render; MDX-specific JSX is **stripped, never executed** (§7.1/7.2). This must be reflected in any user-facing copy — do not market "full MDX component support."
3. **Per-source `contentSelector` mapping** is scheduled inside Phase 5 for 1–2 pilot sources (NestJS, Next.js). It blocks having *real content* to test with, but not the app's architecture; Phases 1–4 use hand-written test bundles.

---

## 2. Phase Plan

Phases are ordered by dependency. Phase 5 (scraper) can run in parallel with Phases 3–4 — it's a standalone tool that never ships inside the Tauri binary (§6).

```
Phase 0  Foundation & hardening
Phase 1  Core shell (PRD M1)
Phase 2  Import pipeline & content security (PRD M2)
Phase 3  Reader UI (PRD M3)          ┐
Phase 4  Search (PRD M4)             ┤ Phase 5 can run in
Phase 5  Scraper CLI (PRD §6)         │ parallel with 3–4
Phase 6  Arch packaging & QA (PRD M5) ┘
```

---

### Phase 0 — Foundation & Hardening

**Goal:** turn the pristine scaffold into a sane, secured baseline before any feature work. (Added phase — the PRD assumes these; the repo doesn't have them yet.)

**Tasks**

- [ ] `git init`, first commit of the scaffold + this plan (`.gitignore` already exists).
- [ ] Wire Tailwind 4 into Vite: add `@tailwindcss/vite` plugin to `vite.config.ts` and `@import "tailwindcss";` to the stylesheet; delete stock `App.css` template styles. (Tailwind is installed but completely unwired.)
- [ ] Register the three plugins in `src-tauri/src/lib.rs` that are declared in `Cargo.toml` but inert: `tauri_plugin_clipboard_manager`, `tauri_plugin_dialog`, `tauri_plugin_fs`.
- [ ] Extend `src-tauri/capabilities/default.json` with the minimum permissions those plugins need (dialog: open; clipboard: write-text; fs: read-only scope limited to the import picker flow). **No shell, no broad fs** — §7.3.
- [ ] Set a strict CSP in `tauri.conf.json` (currently `null`): disallow inline scripts, aggressive `connect-src`/`img-src` — the app is offline at render time, so CSP can be maximal (§7.4). Verify dev mode still works (HMR may need a dev-only CSP override).
- [ ] Window config: resize to ~1200×800, sensible min size, product title "Offline Docs".
- [ ] Remove template cruft: `greet` command, demo `App.tsx` content, `tauri.svg`/`vite.svg` references, template README text.
- [ ] Tooling baseline: ESLint + Prettier (flat config), `tsc --noEmit` in CI-able script; decide on Vitest now (used in Phases 2–3).

**Acceptance**

- `pnpm tauri dev` launches with a Tailwind-styled blank three-region layout; clipboard/dialog/fs commands callable from a test button; CSP blocks injected inline scripts (manual test via devtools).
- Repo is under git with a clean initial commit.

---

### Phase 1 — Core Shell (PRD §13 M1)

**Goal:** SQLite schema, XDG-correct storage, settings persistence, typed command layer, layout skeleton.

**Tasks**

- [ ] **XDG path resolution (§5.4):** resolve `$XDG_DATA_HOME` / `$XDG_CONFIG_HOME` / `$XDG_CACHE_HOME` with the spec's fallbacks. Use the `directories` crate (or Tauri's `app_data_dir()` after verifying it honors `$XDG_DATA_HOME` on Linux — verify, don't assume). Paths: `library.sqlite3`, `settings.json`, `cache/offline-docs/` for import staging.
- [ ] **Migration system:** embedded SQL migrations with a `user_version`/`migrations` table; runs on first launch. Migration 001 = full core schema of §5.1: `docs`, `versions`, `pages` (incl. `parent_id`, `raw_source`, `sort_order`), `bookmarks` (schema only — feature ships later, §10), and the three indices (`idx_versions_doc`, `idx_pages_version`, `idx_pages_parent`).
- [ ] **FTS5 prerequisite check:** confirm the bundled SQLite build in `rusqlite 0.40.2` has FTS5 enabled (`bundled` may not include it — add the required feature flag, e.g. `bundled-full` or the FTS5-enabling combo, and assert `SELECT fts5(?1)` works at startup). Resolves before Phase 4 depends on it.
- [ ] **State & command layer (§4):** `tauri::State` holding `Mutex<Connection>`; typed commands with serde structs: `list_docs`, `list_versions`, `get_page`, `get_settings`, `set_settings`. React talks to the DB **only** through these — no SQLite/WASM binding in the webview, ever (§4 key decision).
- [ ] **Settings (§2 theming, §5.4):** `settings.json` read/write in config dir; keys: `theme` ("system" | "light" | "dark"), `last_opened_doc`. `theme: "system"` is the default.
- [ ] **Frontend layout skeleton (§8):** three-region shell — top bar (global search input placeholder + settings gear), left rail (library list placeholder + Import button placeholder), content pane. Tailwind-styled, no router (navigation is sidebar state, per §3 note) — remove `react-router-dom` if confirmed unused.
- [ ] SQLite pragmas: `journal_mode=WAL`, `foreign_keys=ON`.

**Acceptance**

- Fresh launch creates `library.sqlite3` under the correct XDG path with all tables/indices; deleting `$XDG_DATA_HOME/offline-docs` and relaunching recreates it.
- `get_page`/`list_docs` round-trip via `invoke` from a dev-only debug view; theme setting persists across restart.

---

### Phase 2 — Import Pipeline & Content Security (PRD §6.5–§6.6, §7)

**Goal:** bundles go in, sanitized rows come out, nothing unsafe ever reaches SQLite or the webview. This is the security-critical phase.

**Tasks**

- [ ] **Bundle format (§6.6):** `manifest.json` (`{ doc, slug, version }`) + `pages/NN-slug.md|.mdx`. Parse manifest, walk `pages/` sorted by numeric filename prefix → `sort_order`; front-matter (`title`, `slug`) overrides filename-derived values.
- [ ] **Import validation (§7.5):** reject non-`.md`/`.mdx`/`.json` entries; cap total uncompressed size; if given a `.zip`: extract to the XDG cache staging dir with **path-traversal guards** (reject entry names containing `..` / absolute paths) and **zip-bomb guards** (compression-ratio + total-size checks).
- [ ] **MD/MDX → sanitized HTML at import time (§7.1, §7.2):** Rust-side pipeline — decision: **comrak** (GFM, front-matter extension) → **ammonia** allow-list sanitizer. Sanitizer allow-list: headings, lists, tables, links, images, code fences, blockquotes, inline formatting — everything else stripped; `javascript:` URLs and inline event handlers removed; relative image/URL rewriting to an internal scheme if needed. `raw_source` column keeps the original markdown. *Alternative noted in PRD (remark/rehype in TS or mdxjs-rs) — comrak+ammonia keeps the trust boundary in Rust where the DB lives; document the choice in code.*
- [ ] **Transactional import (§11):** all `docs`/`versions`/`pages` inserts in one SQLite transaction; any malformed file → rollback, nothing persisted; version rows deduped via `UNIQUE(doc_id, version)`.
- [ ] **Commands:** `import_bundle` (folder picker via the dialog plugin; also accept `.zip` path), returning a structured result (imported counts, or error details). `get_page_tree` (nested tree for one version — `parent_id` — selecting **no** `content` column, §11).
- [ ] **Unit tests for the sanitizer** (the §9 "MDX-specific JSX is never executed unsafely" criterion): `<script>`, `onclick=` handlers, `javascript:` hrefs, `<iframe>`, event-handler attributes in MDX JSX — all must come out stripped; safe constructs must survive.

**Acceptance (PRD §9 rows: Import documentation; Markdown/MDX rendering)**

- Import a valid folder/zip → doc appears in sidebar immediately; a malformed bundle (bad manifest, corrupt file, oversized, hostile zip entry name) → clear error surfaced in UI, **zero** rows left in DB.
- A page whose markdown contained scripts/handlers renders inert (verified by sanitizer tests + manual check in devtools).

---

### Phase 3 — Reader UI (PRD §8, §13 M3)

**Goal:** the actual reader: sidebar tree, page rendering, code blocks with copy, dark mode.

**Tasks**

- [ ] **Sidebar (§8):** left rail lists docs → expandable versions → collapsible page tree (`parent_id` nesting) for the selected version. Only one version's tree renders at a time (§9).
- [ ] **Version switcher (§8 note):** dropdown in the content pane header to pick which *imported* version to browse — explicitly not diffing/upgrading (§10).
- [ ] **Page rendering (§7.2, §9):** render `pages.content` (already-sanitized HTML) via `dangerouslySetInnerHTML`; no MDX is ever evaluated as live JSX. Content fetched per page via `get_page` — target **<100ms** from click to rendered (§11; keep list queries free of the `content` column).
- [ ] **Code blocks:** `shiki` with grammars/themes bundled at build time (offline); language inferred from the fenced info string at render time from `raw_source`/code language hints; **Copy** button copies raw code text via `@tauri-apps/plugin-clipboard-manager` (not the browser Clipboard API, per §8). *Size spike here (§12.1): measure artifact size with shiki's bundled grammars; fallback = prism-based highlighting if over budget.*
- [ ] **Dark mode (§2, §9):** default follows GTK `prefer-dark` via Tauri's theme API (`system` setting); manual Light/Dark/System toggle in the settings menu; choice persisted in `settings.json` and survives restarts.
- [ ] Scroll position, active-page highlight in the tree, keyboard-friendly focus states (basic accessibility; full shortcut system is §10 out-of-scope).

**Acceptance (PRD §9 rows: Versions; Sidebar navigation; Dark mode; Code blocks; Markdown/MDX rendering)**

- All §9 rendering checks pass on a test bundle containing headings, lists, tables, links, images, code fences, blockquotes, and JSX-carrying MDX.
- Clicking any page in a 10-doc × multi-version × hundreds-of-pages library renders in <100ms (measure, don't assume).
- Dark mode toggle persists across restarts and defaults to the OS preference.

---

### Phase 4 — Search (PRD §5.2–§5.3, §13 M4)

**Goal:** instant full-text search, global and scoped, powered by FTS5.

**Tasks**

- [ ] **Migration 002 (§5.2):** `pages_fts` external-content FTS5 virtual table (`content='pages'`, `content_rowid='id'`, `tokenize='porter unicode61'`) + the three sync triggers (`pages_ai`, `pages_ad`, `pages_au`). Backfill existing rows.
- [ ] **Query commands (§5.3):** `search_global(query)` — FTS5 `MATCH` + `bm25(pages_fts)` ranking + `snippet(pages_fts, 1, '<mark>', '</mark>', '…', 10)`, `LIMIT 30`, joined to `pages`/`versions`/`docs`; **group results by `doc_name` in the Rust handler** before returning (matches §8 mock). `search_scoped(query, version_id)` — same plus `AND p.version_id = ?2`.
- [ ] Query-safety: sanitize/escape the FTS5 MATCH expression (user-typed `AND`/`OR`/`"`/`NEAR` should not throw SQLite syntax errors — wrap user terms as quoted phrases or escape, with a documented behavior).
- [ ] **Global search UI (§8, §9):** top-bar input, debounced `invoke('search_global', …)` (~200ms), results grouped under each doc's heading with highlighted snippets, click = jump-to-page. **Scoped search UI:** a "Search NestJS docs" input in the open doc's header filtering to that `version_id` only.
- [ ] **Performance (§11):** verify search latency on a large synthetic library (10+ docs × versions × hundreds of pages); indices from §5.1 in place; no `SELECT content` in any list/search path except the snippet (snippet reads from the FTS table, not `pages.content`).

**Acceptance (PRD §9 rows: Global search; Scoped search)**

- Typing in the top bar returns results across **all** imported docs, grouped by doc, ranked by bm25, with `<mark>`-highlighted snippets.
- The in-doc search returns only pages of the currently selected version.
- Both work offline, with the debounce making typing feel instant.

---

### Phase 5 — Scraper CLI (PRD §6 — standalone, runs parallel with Phases 3–4)

**Goal:** a separate TypeScript CLI that produces bundles in the exact §6.6 format. **Never ships inside the Tauri binary; no runtime dependency on the app** beyond the bundle contract.

**Tasks**

- [ ] **Workspace setup:** add `scraper/` as a pnpm workspace package (extend `pnpm-workspace.yaml` with `packages:`). Deps: `playwright`, `turndown`, `turndown-plugin-gfm`. Stays TS-only — **no** scraping framework, no Python (§6.2 decision, with its revisit trigger recorded: dozens of sources + recurring selector breakage).
- [ ] **Types & config (§6.3):** `SourceConfig` interface exactly as PRD (`docSlug`, `version`, `baseUrl`, `contentSelector`, `pages[]` with url/slug/title/order); one explicit config file per library under `sources/` (start with `nestjs.ts`).
- [ ] **Scrape core (§6.3):** Playwright chromium, `waitUntil: "networkidle"`, `contentSelector`-scoped `$eval` → `cleanHtml()` strips `<script>`/`<iframe>`/`<style>` **before** Turndown conversion (sanitization **layer 1 of 2**, §6.4) → GFM-flavored Markdown → `NN-slug.md` with front-matter → `manifest.json` → `pages/` output dir per `{docSlug}/{version}/`.
- [ ] **Failure model (§6.5):** per-page try/catch — one broken selector logs and continues; end-of-run summary of failures; **no** automatic retry/backoff/self-healing (manual runs are human-reviewed by design).
- [ ] **Pilot sources (§12.3):** hand-map `contentSelector` + page lists for **NestJS** and **Next.js**; produce real bundles for app testing.
- [ ] **Round-trip proof (§6.4 layer 2):** a scraped bundle imports through the Phase 2 pipeline **unchanged** — the importer does not and must not special-case scraper output.

**Acceptance**

- `pnpm --filter scraper scrape nestjs` produces `nestjs-v11.bundle/` matching §6.6; failures (e.g. one wrong selector) are reported per-page without aborting the run.
- The scraped bundle imports cleanly and renders in the app; content contains no page chrome (widget/banner residue), proving layer-1 scoping works.

---

### Phase 6 — Arch Packaging & QA (PRD §2, §2.1, §9 "Fully offline", §11, §13 M5)

**Goal:** shippable artifacts for Arch (primary) and generic Linux (AppImage), verified offline on X11 and Wayland.

**Tasks**

- [ ] **Desktop integration (§2):** `.desktop` file (`assets/offline-docs.desktop`) + full icon set — 16/32/48/128/256 px PNGs + SVG — installed to `hicolor`; verify it appears in GNOME, KDE, and a Hyprland/wayland wofi/rofi launcher.
- [ ] **AppImage:** `tauri build --bundles appimage` artifact for non-Arch users.
- [ ] **AUR packages (§2.1):** two `PKGBUILD`s in an `aur/` directory (ready to publish): `offline-docs` (builds from source; PRD's sketch refined — skip bundling for the pure Arch package, install the raw `cargo build --release` binary) and `offline-docs-bin` (installs the prebuilt release from GitHub Releases). `depends=('webkit2gtk-4.1' 'gtk3' 'librsvg')`, `makedepends` per §2. Verify both install and launch in a clean chroot/container.
- [ ] **X11 + Wayland smoke tests (§2):** full manual QA pass under both display servers. Document (do **not** hardcode) the `WEBKIT_DISABLE_COMPOSITING_MODE=1` workaround for WebKitGTK+Wayland+NVIDIA in README/wiki (§2).
- [ ] **Offline verification (§9):** run the app with all egress blocked (network namespace / firewall) and confirm zero functional regression — import from disk, read, search, theme, copy.
- [ ] **Binary size budget (§11):** measure installed size; target well under 20–30 MB; record as a per-release regression metric (CI-able script comparing against a stored baseline). Includes the Phase 3 shiki-spike outcome.
- [ ] **Startup time (§11):** confirm cold start is near-instant (webview + single SQLite open); measure and note the baseline.
- [ ] **Portability audit (§11):** grep for hardcoded `/usr/share`-style paths or Arch assumptions in Rust/React code; all paths must come from XDG/`directories` resolution.
- [ ] **Release engineering:** GitHub Releases with the AppImage + prebuilt binary (feeds `offline-docs-bin`); versioning aligned to `0.1.0` MVP.
- [ ] Rename the product away from the `offline-docs` codename if a final name is chosen before ship (PRD header note).

**Acceptance (PRD §9 row: Fully offline; §2 platform table)**

- App installs via both AUR packages and runs from the AppImage on a clean Arch system.
- Zero egress confirmed under network-blocked namespace with all features working.
- X11 and Wayland sessions both pass the smoke checklist.

---

## 3. Cross-Cutting

### Testing strategy
- **Rust (Phases 1–4):** unit tests for sanitizer allow-list (the security-critical suite), importer (valid/invalid bundles, zip traversal/bomb cases, transactional rollback), FTS query builders, migration idempotency.
- **TypeScript (Phases 3–4):** Vitest for search debounce logic, result grouping, tree building from `parent_id`, theme persistence; component tests kept thin since the trust boundary is Rust-side.
- **Manual QA (Phase 6):** X11/Wayland checklist, offline namespace run, desktop-launcher visibility, clean-chroot AUR installs.

### Out of scope for V1 (PRD §10 — preserved verbatim)
Bookmarks (table exists, no UI) · reading progress · recently viewed · command palette · custom keyboard shortcuts · personal notes · saved snippets · auto doc updates · version diffing/upgrade · automatic doc downloading.

### Risk register

| Risk | Phase | Mitigation |
|---|---|---|
| `rusqlite` bundled SQLite may lack FTS5 | 1 | Explicit feature-flag verification + startup assert before Phase 4 depends on it |
| shiki bundled grammars blow the 20–30 MB size budget | 3 | Size spike with prism fallback already chosen |
| Strict CSP breaks Tauri dev/HMR | 0 | Dev-only CSP override, prod CSP stays maximal |
| Tauri `app_data_dir()` may not honor `$XDG_DATA_HOME` as expected | 1 | Verify in Phase 1; fall back to the `directories` crate |
| FTS5 MATCH syntax errors from raw user input | 4 | Quote/escape user terms with documented behavior |
| `contentSelector` drift on target sites | 5 | Accepted by design (manual runs); revisit trigger per §6.2 |
| WebKitGTK+Wayland+NVIDIA glitches | 6 | Document env-var workaround, never hardcode (§2) |

### Suggested commit/PR cadence
One PR per phase (Phase 5 may split per source config). Each PR must keep `pnpm tauri dev` working and tests green; Phase 6 ends with the `v0.1.0` tag that the AUR `-bin` package points at.
