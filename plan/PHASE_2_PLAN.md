# PHASE 2 — Import Pipeline & Content Security: Detailed Plan

**Parent:** [`MAIN_PLAN.md`](./MAIN_PLAN.md) → Phase 2 (PRD §13 M2)
**Goal:** bundles in → sanitized rows in SQLite → nothing else. The security-critical phase: every line of untrusted content passes through here exactly once, at import time (PRD §7.1), never at render time.
**PRD references:** §6.5 (Import action), §6.6 (bundle format), §6.4 (layer-2 sanitization), §7.1/7.2 (compile-then-store, never eval), §7.5 (validation), §11 (transactional reliability).
**Prerequisite:** Phase 1 complete (migrated schema, `AppState`, `AppError`, `paths.cache_dir`, typed command layer).
**Estimated scope:** 3–4 working sessions — the largest Rust phase.

---

## 1. Starting State (after Phase 1)

- `pages` table exists with `content`/`raw_source`/`sort_order`/`parent_id` — all NULL-content, awaiting an importer. `docs.icon` exists but no bundle format defines an icon yet (§6.6 manifest has none) — stays NULL in V1.
- Command pattern (async + `spawn_blocking`), DTO conventions, and `src/lib/api.ts` in place.
- Import button in the sidebar is a non-functional placeholder.
- **No markdown/HTML crates in `Cargo.toml` yet** — this phase adds them.

---

## 2. Decisions to Lock In

1. **New dependencies (Rust):** `comrak` (GFM markdown → HTML, front-matter extension), `ammonia` (allow-list HTML sanitizer), `zip` (zip bundle support), `serde_yaml` (front-matter `title`/`slug` — small, only used on ≤2 keys; not worth a hand-rolled parser to maintain), `walkdir` (bundle folder walk), `thiserror` (already from Phase 1).
2. **Import-time pipeline order (fixed, per page file):**
   `read bytes → (zip: staged extraction) → strip front-matter → comrak → ammonia → store HTML in content, original markdown (minus front-matter? no — **with** front-matter stripped) in raw_source`.
   Sanitize **before** storage, so even a buggy renderer can't leak unsanitized HTML from the DB (§7.1 "already-safe HTML" is a stored property, not a render-time promise).
3. **Re-import semantics: replace the version.** Importing a bundle whose `(doc, version)` already exists runs entirely inside the same transaction: update `docs`/`versions` metadata, `DELETE` the old pages of that version (cascades through FTS triggers in Phase 4), insert fresh ones. Rationale: the scraper workflow (§6.5) is "fix selector, re-run, re-import" — rejecting duplicates would fight the intended loop. Documented in the import result.
4. **`is_default`:** first-imported version of a doc becomes `is_default = 1`; later imports don't steal the flag. No UI for changing it in V1 (PRD §8's version switcher covers selection at runtime).
5. **Bundle size caps (constants in one module, §7.5):** total uncompressed ≤ **500 MB**, single file ≤ **50 MB**, page count ≤ **5 000**, zip compression ratio ≤ **100:1**. Checked *before* allocation, not after.
6. **Internal links: left as-is, made inert at render.** Relative links between pages (`./controllers.md` or `../nav`) are **not** rewritten during import (the target slug map is per-version and brittle to rewrite at this layer). The Phase 3 renderer intercepts clicks on internal links; until then the strict CSP + `opener` capability scoping keeps them harmless. `ammonia` keeps `href` only for `http`/`https`/relative — `javascript:`/`data:` are dropped by its URL filtering.
7. **MDX strategy (resolves §12.2 concretely):** comrak treats MDX's JSX as raw HTML blocks/inline — which means **ammonia's allow-list strips every JSX tag** (`<Button onClick={…}>`, `<Tabs>`, `<Tabs.Tab>`). Code fences containing JSX survive untouched (comrak escapes code content; ammonia doesn't touch it). Result: MDX files render as their markdown skeleton — exactly the §12.2 "Markdown + curated safe components" stance, with **zero extra MDX code** to write or audit. Component rendering is a post-V1 capability add, never an execution path.

---

## 3. Task Breakdown (ordered)

### Task 1 — Bundle model + staging (§6.6, §7.5)
- `src-tauri/src/import/bundle.rs`:
  - `struct BundleManifest { doc: String, slug: String, version: String }` — strict serde (`deny_unknown_fields`): a manifest with a missing `slug` fails the import with a *named-field* error, not a generic parse error.
  - `struct PageFile { order: u32, slug: String, title: Option<String>, format: Md|Mdx, raw: String }` — parsed from filenames `NN-slug.md|.mdx`; the numeric prefix is mandatory (§6.6) and becomes `sort_order`; a duplicate `NN` or duplicate `slug` within one bundle = validation error.
- `src-tauri/src/import/validate.rs`:
  - Allowed extensions: `.md`, `.mdx`, `.json` (manifest only). Anything else in `pages/` → error listing the offending path.
  - Size caps enforced while walking/streaming (Decision 5).
- `src-tauri/src/import/staging.rs` — zip path only:
  - Extract to `<cache_dir>/import/<random>/` with **path-traversal guard**: reject entry names that are absolute or contain `..` segments *before* touching the disk (the §7.5 `../../etc/passwd` case).
  - **Zip-bomb guard:** read entries with an explicit decompressed-bytes budget (total + per-file) and abort on ratio breach — never decompress-then-check.
  - Stage dir is deleted in a drop-guard on every exit path (success, error, panic via `Drop`), and a startup sweep clears stale `<cache_dir>/import/*` older than a day.
- Folder bundles skip staging and are read in place.

### Task 2 — Front-matter + markdown compile (§6.6, §7.1)
- `src-tauri/src/import/compile.rs`:
  - Front-matter: only a leading `---` block; extract `title` / `slug` (overrides filename-derived values); unknown keys ignored; malformed YAML → clear per-file error.
  - comrak options: GFM (tables, strikethrough, task lists — needed for §9 rendering criteria), `frontmatter_extension` on, safe mode irrelevant (ammonia follows), heading anchors **off** (not needed V1).
  - The fenced code info string (` ```ts `) is preserved in the HTML `class="language-ts"` — Phase 3's shiki reads it from there *and* from `raw_source`.

### Task 3 — Sanitizer (§7.1/7.2 — the security core)
- `src-tauri/src/import/sanitize.rs`: one `ammonia::Builder` constructed once, allow-list = §9's rendering set:
  - **Tags:** headings h1–h6, p, a, img, ul/ol/li, blockquote, pre, code, table/thead/tbody/tr/th/td, hr, br, strong, em, del, sup/sub, span, details/summary, figure/figcaption, input (checkbox, disabled — GFM task lists).
  - **Attributes:** `a: href, title`; `img: src, alt, title, width, height`; `code/pre: class`; `th/td: align`; `span: class`; nothing global except `id` on headings (anchor targets) — **no `style`, no `class` from input except code/spans** (blocks scraped-page CSS leakage, §6.4).
  - **URL schemes:** ammonia defaults handle `http/https/relative`; add explicit `link_rel="noopener noreferrer"`.
  - Wrap it in a pure function `sanitize(html: &str) -> String` so the test suite can be exhaustive (Task 6).

### Task 4 — Importer transaction (§6.5, §11)
- `src-tauri/src/import/importer.rs`: `import_bundle(conn, bundle_root) -> ImportResult`:
  1. Validate manifest → compile+sanitize all pages **first, collecting per-file errors** (fail-fast on the first error loses the diagnostics the §9 "errors surfaced clearly" criterion wants; collect all, then fail if any).
  2. `BEGIN IMMEDIATE` → upsert `docs` by slug → upsert `versions` (Decision 3/4) → `DELETE` old pages of that version → insert pages (`parent_id = NULL` for V1 — the bundle format has no hierarchy yet; the column exists and Phase 5 configs may add `parent` in a manifest v2, flagged as a future contract change, not smuggled in now) → update `COMMIT` / `ROLLBACK` on any error.
  3. `ImportResult { doc_name, version, pages_imported, replaced: bool }`.
- **No user-facing partiality is possible:** the transaction covers all row writes; a failure leaves the previous import (if any) fully intact.

### Task 5 — Commands + frontend wiring (§6.5)
- `import_bundle()` command: invokes the **dialog** plugin Rust-side (`rfd`-style via `tauri_plugin_dialog::DialogExt` blocking file picker — *filter: folders + `.zip`*), runs the importer in `spawn_blocking`, returns `ImportResult` or `AppError` with the collected per-file error list.
- Frontend:
  - `src/lib/api.ts`: `importBundle(): Promise<ImportResult>`.
  - Sidebar Import button → calls `importBundle()` → on success: refresh `list_docs` and navigate to the new doc; on failure: an inline error panel (grouped per-file errors, copyable) — **not** a bare alert.
  - Import button gets a busy state (spinning indicator; import of a big bundle must not look frozen).
  - Vitest: error-panel renders grouped errors; success path refreshes the doc list (api.ts mocked).

### Task 6 — Sanitizer + importer test suite (the deliverable's security proof)
- `sanitize` (pure function, exhaustive table tests):
  - stripped: `<script>`, `<iframe>`, `<style>`, `<object>`, `<embed>`, `onclick`/`onerror`/any `on*`, `javascript:` href/src, `data:text/html` href, MDX JSX (`<Button onClick={…}>`, `<Tabs.Tab>`), inline `style=` attributes, `srcset`.
  - preserved: headings, lists, tables, blockquotes, fenced code (incl. info class), links (http/https/relative), images (`data:` images allowed for `img` only), task-list checkboxes.
- `importer`:
  - valid bundle → rows correct (`sort_order` from prefix, front-matter overrides title/slug, raw_source kept).
  - re-import same bundle → replacement (no duplicate versions).
  - malformed manifest / missing manifest / non-`.md` file / duplicate slug / oversized file / hostile zip entry (`../../x.md`) / zip bomb ratio → error, and **assert zero rows persist** after each.
- `staging`: traversal-entry rejection unit test (constructed zip in `tempfile`), stale-cache sweep.

---

## 4. Verification Checklist (acceptance)

Automated:
- [ ] Full Task 6 suite green; `cargo clippy -D warnings` clean.
- [ ] Sanitizer round-trip property: for every fixture, `sanitize(compile(md))` contains no `<script`, no ` on*=` handlers, no `javascript:` (grep-based test over all fixtures).

Manual (MAIN_PLAN Phase 2 acceptance, made concrete):
- [ ] Import a valid folder bundle → doc appears in sidebar immediately with correct page order and titles.
- [ ] Import the same bundle as `.zip` → identical result; re-import → replaces, no duplicates.
- [ ] Malformed bundle (bad manifest / stray `.exe` in `pages/` / truncated page file) → error panel lists the exact file(s); DB state unchanged (verify via `sqlite3` CLI before/after).
- [ ] Hostile zip (path traversal + bomb fixtures from Task 6) rejected with no extraction side effects in the cache dir.
- [ ] A page whose markdown embedded `<script>alert(1)</script>` and `<img onerror=…>` renders inert in the app (devtools: elements absent, CSP silent) — first live cross-check of Phase 0's CSP with Phase 2's sanitizer.

## 5. Handoff to Phase 3

The DB now holds real sanitized HTML; Phase 3 only ever `dangerouslySetInnerHTML`s `pages.content` — it must add **no** sanitization of its own (that would be a second, divergent trust decision) and no `eval`/component-mapping of MDX (§7.2 stands). `get_page_tree` (nested, content-free) is Phase 3's first Rust addition since the bundle format carries no hierarchy yet — sidebar starts as a flat, sort-ordered list. Phase 5's scraper is validated against *this* phase's contract unchanged (§6.4 layer-2 round-trip).
