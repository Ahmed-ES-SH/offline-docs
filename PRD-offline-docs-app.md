# PRD — Offline Documentation Desktop App

**Codename:** `offline-docs` (rename before ship)
**Platform target:** Linux, primary distro **Arch Linux** (pacman/AUR), secondary generic Linux via AppImage
**Author:** Ahmed
**Status:** Draft v1 — MVP scope

---

## 1. Problem & Goal

Developers frequently need to read framework/library documentation (NestJS, Next.js, React, TypeScript, Docker, Node.js, etc.) without a reliable internet connection, or want a single fast local reference instead of juggling browser tabs. There is no lightweight, native, offline-first, full-text-searchable docs reader for Linux.

**Goal:** Ship a native desktop app that stores documentation locally in SQLite, renders it from Markdown/MDX, and provides instant full-text search — both globally and scoped to a single doc/version — with zero network dependency at read time.

**Non-goals for V1:** live doc updates, bookmarks/notes, telemetry, cloud sync, Windows/macOS polish (Tauri gives us portability for free later, but QA effort is Linux-only for V1).

---

## 2. Target Platform Details (Arch Linux specifics)

This section exists because "Linux" is not one target — packaging and runtime behavior differ by distro and display server.

| Concern | Decision |
|---|---|
| Primary package format | **AUR package** (`offline-docs` / `offline-docs-bin`) |
| Secondary format | `.AppImage` (portable, for non-Arch Linux users) |
| Build system | `cargo` + `pnpm`/`npm` via `tauri build` |
| WebView engine | WebKitGTK — **Tauri v2 requires `webkit2gtk-4.1`**, not the older 4.0 package |
| Display server | Must work under both **X11 and Wayland**. Note known WebKitGTK+Wayland+NVIDIA compositing glitches — document the `WEBKIT_DISABLE_COMPOSITING_MODE=1` env-var workaround in the README/wiki, don't hardcode it |
| Runtime deps (pacman) | `webkit2gtk-4.1`, `gtk3`, `libayatana-appindicator` (only if tray icon shipped later), `librsvg` |
| Build-time deps (pacman) | `base-devel`, `rust`, `nodejs`, `pnpm` (or `npm`), `patchelf` |
| Data location | XDG Base Directory spec — see §5.4 |
| Theming | Dark mode should default to following GTK's `prefer-dark` setting via Tauri's theme API, with a manual override stored in app settings |
| Desktop integration | Ship a `.desktop` file + icon set (16/32/48/128/256 px, plus an SVG) so it appears correctly in Arch DEs (GNOME, KDE, Hyprland/waybar launchers, etc.) |

### 2.1 AUR packaging approach

Two AUR packages, standard convention:

- `offline-docs` — builds from source (`cargo`/`pnpm build` inside `PKGBUILD`). This is the one power users and the AUR community expect.
- `offline-docs-bin` — installs a prebuilt release binary from GitHub Releases (faster install, no compile step).

```bash
# PKGBUILD (source variant) — sketch, not final
pkgname=offline-docs
pkgver=0.1.0
pkgrel=1
pkgdesc="Offline-first documentation reader for developers"
arch=('x86_64' 'aarch64')
url="https://github.com/<you>/offline-docs"
license=('MIT')
depends=('webkit2gtk-4.1' 'gtk3' 'librsvg')
makedepends=('rust' 'nodejs' 'pnpm' 'patchelf' 'base-devel')
source=("$pkgname-$pkgver.tar.gz::https://github.com/<you>/offline-docs/archive/v$pkgver.tar.gz")
sha256sums=('SKIP')

build() {
  cd "$pkgname-$pkgver"
  pnpm install --frozen-lockfile
  pnpm tauri build --bundles appimage
}

package() {
  cd "$pkgname-$pkgver"
  install -Dm755 "src-tauri/target/release/$pkgname" "$pkgdir/usr/bin/$pkgname"
  install -Dm644 "assets/$pkgname.desktop" "$pkgdir/usr/share/applications/$pkgname.desktop"
  install -Dm644 "assets/icons/256x256.png" "$pkgdir/usr/share/icons/hicolor/256x256/apps/$pkgname.png"
}
```

**Note:** `tauri build` on Arch may need `WEBKIT_DISABLE_COMPOSITING_MODE` unset at build time and `--bundles appimage` is only needed if you want an AppImage artifact too — for the pure Arch package you can skip bundling entirely and just install the raw binary produced by `cargo build --release` under `src-tauri/target/release/`.

---

## 3. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Shell / native runtime | **Tauri 2.x** (Rust) | Small binary size vs Electron; native webview = fewer bundled deps on Linux |
| UI | **React 18 + TypeScript + Vite** | Vite for fast dev server + HMR inside Tauri's dev shell |
| Local DB | **SQLite** via `rusqlite` or `sqlx` (Rust side) | DB lives on the Rust side; React talks to it only through Tauri `invoke` commands — never a direct SQLite binding in the webview |
| Full-text search | **SQLite FTS5** | External-content FTS table synced via triggers (see §5) |
| Content format | Markdown / MDX | Compiled to sanitized HTML **before** it's ever stored/rendered — see §7 (Security) |
| Styling | Tailwind CSS (matches your existing stack/preferences) | |
| Syntax highlighting | `shiki` (better token fidelity, works offline once grammars are bundled) or `react-syntax-highlighter` with `prism` | `shiki` needs its grammar/theme JSON bundled at build time so highlighting works with zero network |

> Confirms the original spec: no Next.js. Next.js is a web framework (SSR/routing/server actions) — none of that applies inside a Tauri webview. Vite + React Router (or no router at all, since navigation is just sidebar state) is the correct choice.

---

## 4. Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                        Tauri Shell                           │
│  ┌─────────────────────────┐   IPC (invoke/emit)             │
│  │   React + TypeScript     │◄──────────────┐                │
│  │   (webview, no direct    │                │                │
│  │    filesystem/DB access) │                │                │
│  └─────────────────────────┘                │                │
│                                               ▼                │
│                                   ┌───────────────────────┐   │
│                                   │   Rust Core (backend)  │   │
│                                   │                        │   │
│                                   │  - Tauri commands API   │   │
│                                   │  - SQLite access layer  │   │
│                                   │  - MD/MDX importer      │   │
│                                   │    + sanitizer          │   │
│                                   │  - FTS5 query builder    │   │
│                                   └──────────┬─────────────┘   │
│                                              │                 │
│                                              ▼                 │
│                              ┌───────────────────────────┐    │
│                              │  SQLite file (single file) │    │
│                              │  ~/.local/share/offline-   │    │
│                              │  docs/library.sqlite3      │    │
│                              └───────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**Key architectural decision:** all SQLite access happens in Rust, exposed to React via typed Tauri commands (`invoke('search_global', { query })`, `invoke('get_page', { pageId })`, etc.). React never imports a SQLite/WASM binding directly. This keeps the trust boundary in one place and matches Tauri's own security model (webview = untrusted-ish surface, Rust core = trusted).

---

## 5. Data Model

### 5.1 Core schema

```sql
CREATE TABLE docs (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  name        TEXT NOT NULL UNIQUE,        -- "NestJS", "Next.js"
  slug        TEXT NOT NULL UNIQUE,        -- "nestjs", "nextjs"
  description TEXT,
  icon        TEXT,                        -- path or bundled icon key
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE versions (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  doc_id      INTEGER NOT NULL REFERENCES docs(id) ON DELETE CASCADE,
  version     TEXT NOT NULL,               -- "v10", "v11", "latest"
  is_default  INTEGER NOT NULL DEFAULT 0,  -- which version opens by default
  imported_at TEXT NOT NULL DEFAULT (datetime('now')),
  UNIQUE(doc_id, version)
);

CREATE TABLE pages (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  version_id  INTEGER NOT NULL REFERENCES versions(id) ON DELETE CASCADE,
  parent_id   INTEGER REFERENCES pages(id) ON DELETE CASCADE, -- for nested sidebar sections
  title       TEXT NOT NULL,
  slug        TEXT NOT NULL,               -- "controllers"
  content     TEXT NOT NULL,               -- sanitized HTML (compiled from MD/MDX)
  raw_source  TEXT,                        -- original markdown, kept for re-render/debug
  sort_order  INTEGER NOT NULL DEFAULT 0,
  UNIQUE(version_id, slug)
);

CREATE TABLE bookmarks (          -- table exists in schema now, feature ships later
  id       INTEGER PRIMARY KEY AUTOINCREMENT,
  page_id  INTEGER NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
  note     TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_versions_doc      ON versions(doc_id);
CREATE INDEX idx_pages_version     ON pages(version_id);
CREATE INDEX idx_pages_parent      ON pages(parent_id);
```

### 5.2 Full-text search (FTS5, external-content table)

Use **external content** FTS5 so the indexed text isn't duplicated in the DB file, and keep it in sync with triggers:

```sql
CREATE VIRTUAL TABLE pages_fts USING fts5(
  title,
  content,
  content='pages',
  content_rowid='id',
  tokenize = 'porter unicode61'
);

-- Keep FTS in sync
CREATE TRIGGER pages_ai AFTER INSERT ON pages BEGIN
  INSERT INTO pages_fts(rowid, title, content) VALUES (new.id, new.title, new.content);
END;

CREATE TRIGGER pages_ad AFTER DELETE ON pages BEGIN
  INSERT INTO pages_fts(pages_fts, rowid, title, content) VALUES ('delete', old.id, old.title, old.content);
END;

CREATE TRIGGER pages_au AFTER UPDATE ON pages BEGIN
  INSERT INTO pages_fts(pages_fts, rowid, title, content) VALUES ('delete', old.id, old.title, old.content);
  INSERT INTO pages_fts(rowid, title, content) VALUES (new.id, new.title, new.content);
END;
```

### 5.3 Query patterns

**Global search** (across every doc/version):

```sql
SELECT
  d.name AS doc_name,
  v.version,
  p.title,
  p.slug,
  snippet(pages_fts, 1, '<mark>', '</mark>', '…', 10) AS excerpt,
  bm25(pages_fts) AS rank
FROM pages_fts
JOIN pages p ON p.id = pages_fts.rowid
JOIN versions v ON v.id = p.version_id
JOIN docs d ON d.id = v.doc_id
WHERE pages_fts MATCH ?1
ORDER BY rank
LIMIT 30;
```

**Scoped search** (inside one doc+version only) — same query with an added filter:

```sql
WHERE pages_fts MATCH ?1
  AND p.version_id = ?2
ORDER BY rank
LIMIT 30;
```

Group the global results by `doc_name` in the Rust command handler (or in React) before returning, to match the UI mock in the spec (results nested under each doc's heading).

### 5.4 Storage locations (XDG Base Directory spec)

Don't hardcode `~/.local/share`; resolve it (Tauri's `app_data_dir()` / the Rust `directories` crate already does this correctly per-OS):

| Data | Path |
|---|---|
| SQLite DB | `$XDG_DATA_HOME/offline-docs/library.sqlite3` (fallback `~/.local/share/offline-docs/`) |
| App settings (theme, last-opened doc) | `$XDG_CONFIG_HOME/offline-docs/settings.json` (fallback `~/.config/offline-docs/`) |
| Temp/import staging | `$XDG_CACHE_HOME/offline-docs/` (fallback `~/.cache/offline-docs/`) |

---

## 6. Documentation Ingestion & Scraping Pipeline (V1 scope clarification)

The original spec lists "Automatic documentation downloading" as a **future** feature — which means V1 needs a defined, *manual* way to get docs into the app. This section is the resolved design for that pipeline: where content comes from, how it's extracted, and in what shape it reaches the app. This is a **separate, standalone tool** from the desktop app itself — it never ships inside the Tauri binary and has no runtime dependency on it beyond producing bundles in the agreed format (§6.5).

### 6.1 Content source priority

Content source is decided **per library**, not globally, using this priority order:

1. **Raw Markdown/MDX from a public GitHub repo**, when the library's docs are actually authored there (e.g. many projects keep `docs/` as the single source of truth for their site). Cleanest option — no HTML to strip, structure/ordering often already defined in a nav/meta file.
2. **A dedicated Markdown endpoint on the official site**, when available (`.md` suffix on a doc URL, or an `llms.txt`/`llms-full.txt` file at the site root — increasingly common).
3. **Scraping the rendered documentation site directly.** In practice this is the primary path for many libraries, not a fallback — not every project publishes its docs as raw source, so the pipeline treats this as a first-class case rather than an edge case.

### 6.2 Decision: a small TypeScript scraper, not a dedicated scraping framework

A general-purpose scraping framework (e.g. Python's Scrapling, which offers adaptive/self-healing selectors and anti-bot bypass) was evaluated and deliberately **not** adopted for V1. Reasoning:

| Factor | Assessment |
|---|---|
| Anti-bot protection on target sites | Not expected — these are public developer documentation sites, not e-commerce/aggregator targets |
| Number of sources | Fixed and known upfront (NestJS, Next.js, React, TypeScript, Docker, Node.js, …) — not open-ended crawling |
| Run frequency | Manual/periodic (run by hand when refreshing a doc set), not continuous/scheduled — a human reviews output each run |
| Structural drift risk | Official doc sites change layout rarely (roughly yearly); a broken selector is caught and fixed in minutes during a manual run |
| Toolchain cost | Adding a scraping framework in Python means a second runtime/toolchain alongside the existing Rust (Tauri) + TypeScript (React/Vite) stack, for a problem (adaptive selector healing, anti-bot bypass) this project doesn't actually have |
| HTML → Markdown conversion | Not provided out of the box by general scraping frameworks; still needs a separate conversion step regardless of which fetcher is used |

**Conclusion:** a small TypeScript CLI using Playwright (fetch/render) + Turndown (HTML → Markdown) is sufficient, keeps the entire project in two languages (Rust + TypeScript) instead of three, and matches the actual failure mode of this problem (occasional, human-reviewed selector breakage) rather than the failure mode general scraping frameworks are built for (continuous scraping against adversarial, frequently-changing targets).

**Revisit trigger:** reconsider a dedicated adaptive-scraping framework only if the source count grows into the dozens *and* selector breakage becomes a recurring, measurable maintenance cost. Not a V1 concern.

### 6.3 Scraper architecture

One config object per documentation source (`SourceConfig`), listing its base URL, the CSS selector that isolates the actual content region, and the explicit list of pages to fetch (URL, slug, title, sidebar order). No global "crawl everything" logic — each source is walked explicitly, because sidebar/nav structure differs per site and is more reliably hand-mapped once than auto-discovered per run.

```typescript
// scrapers/types.ts
export interface SourceConfig {
  docSlug: string;
  version: string;
  baseUrl: string;
  contentSelector: string;   // e.g. "article.markdown-body" — scoped to the actual content region only
  pages: { url: string; slug: string; title: string; order: number }[];
}
```

```typescript
// scrapers/scrape.ts
import { chromium, type Browser } from "playwright";
import TurndownService from "turndown";
import { gfm } from "turndown-plugin-gfm";
import fs from "node:fs/promises";
import path from "node:path";
import type { SourceConfig } from "./types";

const turndown = new TurndownService({ codeBlockStyle: "fenced" });
turndown.use(gfm);

// Strip non-content elements before conversion — see §6.4, this is layer 1 of 2
function cleanHtml(html: string): string {
  return html
    .replace(/<script[\s\S]*?<\/script>/gi, "")
    .replace(/<iframe[\s\S]*?<\/iframe>/gi, "")
    .replace(/<style[\s\S]*?<\/style>/gi, "");
}

async function scrapePage(browser: Browser, url: string, selector: string): Promise<string> {
  const page = await browser.newPage();
  try {
    await page.goto(url, { waitUntil: "networkidle", timeout: 30_000 });
    const html = await page.$eval(selector, (el) => el.innerHTML).catch(() => {
      throw new Error(`Selector "${selector}" not found on ${url}`);
    });
    return turndown.turndown(cleanHtml(html));
  } finally {
    await page.close();
  }
}

export async function scrapeSource(config: SourceConfig, outRoot: string) {
  const browser = await chromium.launch();
  const outDir = path.join(outRoot, config.docSlug, config.version, "pages");
  await fs.mkdir(outDir, { recursive: true });

  const failures: string[] = [];

  for (const p of config.pages) {
    try {
      const markdown = await scrapePage(browser, p.url, config.contentSelector);
      const filename = `${String(p.order).padStart(2, "0")}-${p.slug}.md`;
      const frontmatter = `---\ntitle: "${p.title}"\nslug: "${p.slug}"\n---\n\n`;
      await fs.writeFile(path.join(outDir, filename), frontmatter + markdown, "utf-8");
    } catch (err) {
      failures.push(`${p.url}: ${(err as Error).message}`);
    }
  }

  await browser.close();

  await fs.writeFile(
    path.join(outRoot, config.docSlug, config.version, "manifest.json"),
    JSON.stringify({ doc: config.docSlug, version: config.version }, null, 2)
  );

  if (failures.length) {
    console.warn(`${failures.length} page(s) failed:\n${failures.join("\n")}`);
  }
}
```

```typescript
// sources/nestjs.ts — one explicit config file per library
import type { SourceConfig } from "../scrapers/types";

export const nestjsSource: SourceConfig = {
  docSlug: "nestjs",
  version: "v11",
  baseUrl: "https://docs.nestjs.com",
  contentSelector: "article.markdown-body",
  pages: [
    { url: "https://docs.nestjs.com/introduction", slug: "introduction", title: "Introduction", order: 0 },
    { url: "https://docs.nestjs.com/controllers", slug: "controllers", title: "Controllers", order: 1 },
    // ...
  ],
};
```

### 6.4 Sanitization requirements are elevated for scraped content

Content sourced from a live external website is **less trusted** than content pulled from a project's own GitHub repo — live sites can carry tracking scripts, ad widgets, cookie-consent banners, or embedded chat widgets that aren't part of the actual documentation. This means sanitization is defense-in-depth, applied at two separate layers, not one:

1. **At scrape time** (`cleanHtml()` in §6.3): strip `<script>`, `<iframe>`, `<style>` before HTML-to-Markdown conversion, and keep `contentSelector` scoped as tightly as possible (`main article`, not `body`) so unrelated page chrome never enters the pipeline at all.
2. **At import time** (§7, unchanged and still mandatory): the app's own MD/MDX → sanitized-HTML compiler runs on every bundle regardless of source, including bundles produced by this scraper. The import-time sanitizer is never skipped or trusted-sourced — a scraped bundle and a hand-written bundle go through the identical import path.

### 6.5 Error handling & resilience

The scraper fails **per page, not per run** — one broken selector logs a failure and the rest of the source continues, with a summary report printed at the end. Because runs are manual and human-reviewed (§6.2), there is no automatic retry, backoff, or self-healing selector logic; a human updates the one broken `contentSelector` and re-runs, which is faster and more predictable than debugging an automatic recovery heuristic.

### 6.6 Output: bundle format (target of the scraper, input to the app)

The scraper's output is the same bundle format the app's importer already expects — the scraper is simply one producer of this format, and hand-curated bundles remain equally valid:

```text
nestjs-v11.bundle/
├── manifest.json        # { "doc": "NestJS", "slug": "nestjs", "version": "v11" }
└── pages/
    ├── 00-introduction.md
    ├── 01-controllers.md
    ├── 02-providers.mdx
    └── ...
```

- App exposes an **"Import Documentation"** action (file picker → select a folder or a `.zip` of the above structure).
- Rust-side importer: parses `manifest.json`, walks `pages/`, compiles each file (see §7 — full sanitization always runs here too, per §6.4), inserts `docs`/`versions`/`pages` rows in a transaction, fails atomically if any file is malformed.
- Filename prefix (`00-`, `01-`) drives `sort_order`; front-matter in each `.md`/`.mdx` file can override `title`/`slug` if present.
- This keeps the app itself fully offline and decoupled from the scraper — the import contract doesn't change regardless of whether a bundle came from GitHub source, a Markdown endpoint, or the TypeScript scraper in §6.3.

---

## 7. Rendering & Security

This is the part of the spec that most needs hardening before writing code, because it's a real vulnerability class, not a theoretical one.

**Risk:** Tauri apps have a Rust backend reachable from the webview via `invoke()`. If imported documentation content can execute arbitrary JavaScript in that webview (e.g., an MDX file with a `<script>` or an `onClick` JSX handler), that script runs with access to whatever Tauri commands your app exposes — this is a realistic path to local file read/write or worse, not just an XSS annoyance.

**Mitigations (all required for V1, not "nice to have"):**

1. **Compile MD/MDX to sanitized HTML at import time**, not at render time. Use `remark` → `rehype` → `rehype-sanitize` (or `mdxjs-rs`'s Rust pipeline if you compile on the Rust side) so `pages.content` in SQLite is already-safe HTML with `<script>`, inline event handlers, and `javascript:` URLs stripped.
2. **Never `eval` MDX as live JSX** in the renderer for imported content — render it as static sanitized HTML via `dangerouslySetInnerHTML` only *after* sanitization, or better, parse to a safe AST and render with a strict allow-list of React components (headings, code blocks, tables, links, images) instead of arbitrary components.
3. **Lock down Tauri's capability/permission system (v2)** — only allow the specific commands the docs viewer needs (`search_global`, `search_scoped`, `get_page`, `list_docs`, `import_bundle`). No filesystem/shell capabilities exposed beyond what import needs, and definitely nothing reachable from a webview context loading imported content.
4. **Set a Content-Security-Policy** in `tauri.conf.json` disallowing inline scripts and restricting `connect-src`/`img-src` appropriately (the app is offline — CSP can be aggressive here since there's no legitimate external network call at render time).
5. **Validate/limit import bundle size and file types** during import (reject non-`.md`/`.mdx`/`.json` files, cap total size) to avoid zip-bomb or path-traversal issues (`../../etc/passwd` inside a zip entry name).

---

## 8. UI / UX

Matches the spec's layout, refined:

```text
┌───────────────────────────────────────────────────────────────┐
│ 🔍 Search all documentation                              [⚙]  │
├───────────────┬───────────────────────────────────────────────┤
│  Library       │  NestJS  v11 ▾                                │
│  ─────────     │  🔍 Search NestJS docs                        │
│  ▸ NestJS      │                                                │
│  ▸ Next.js     │  Introduction                                  │
│  ▸ React       │  Controllers                                   │
│  ▸ TypeScript  │  Providers                                     │
│  ▸ Docker      │  Modules                                       │
│  ▸ Node.js     │  ...                                           │
│                │  ──────────────────────────────────────────── │
│  [+ Import]    │  <rendered page content, code blocks, etc.>    │
└───────────────┴───────────────────────────────────────────────┘
```

- Left rail: list of imported docs → expands to versions → expands to page tree (uses `parent_id` for nesting).
- Top bar: global search input; typing triggers debounced `invoke('search_global', ...)`, results grouped by doc name with jump-to-page.
- Content pane: version switcher dropdown (still V1 — the spec lists "version switching" as future, but a doc with 2 imported versions needs *some* way to pick one; treat this as "select which imported version to browse," not the more complex "diff between versions" feature implied later).
- Dark mode toggle in settings, defaulting to OS preference.
- Code blocks: syntax-highlighted, with a "Copy" button (clipboard via Tauri's `clipboard` plugin, not the browser Clipboard API, for permission consistency).

---

## 9. V1 Feature Scope (acceptance criteria)

| Feature | Acceptance criteria |
|---|---|
| Import documentation | User can pick a folder/zip matching the bundle format (§6) and see it appear in the sidebar after import, with errors surfaced clearly on malformed bundles |
| Versions | A doc can have 2+ versions; sidebar shows them; only one page tree renders at a time per selected version |
| Markdown/MDX rendering | Headings, lists, tables, links, images, code fences, blockquotes render correctly; MDX-specific JSX is either safely allow-listed or stripped, never executed unsafely |
| Sidebar navigation | Nested pages (`parent_id`) render as collapsible tree; clicking a page loads it in <100ms from local SQLite |
| Global search | Typing a query returns results across **all** imported docs, grouped by doc, ranked by FTS5 `bm25`, with a highlighted snippet |
| Scoped search | Same search UI inside an open doc filters to that `version_id` only |
| Dark mode | Toggle persists across restarts (stored in settings.json); defaults to OS `prefer-dark` |
| Code blocks | Syntax highlighting per language (inferred from fenced code info string); "Copy" button copies raw code, not rendered HTML |
| Fully offline | App makes zero network requests after install; verified by running with network namespace/firewall blocking all egress and confirming no functional regression |

---

## 10. Explicitly Out of Scope for V1

Directly from the spec, preserved as-is:

- Bookmarks (table exists in schema, feature UI does not)
- Reading progress
- Recently viewed pages
- Command palette
- Custom keyboard navigation shortcuts
- Personal notes
- Saved code snippets
- Automatic documentation update checking
- In-app version switching *with diffing* (basic "pick which imported version" is in scope, per §8; version *upgrade/refresh* is not)
- Automatic documentation downloading from the internet

---

## 11. Non-Functional Requirements

- **Performance:** page load from SQLite → render should feel instant (<100ms) even with a library containing 10+ docs × multiple versions × hundreds of pages each. Add indices as in §5.1; avoid `SELECT *` on `pages.content` in list views (fetch `content` only when opening a page).
- **Binary size:** Tauri should keep this well under 20–30MB installed, a key advantage over an Electron equivalent — track this as a regression metric per release.
- **Startup time:** cold start should be near-instant since there's no server/runtime bootstrap beyond the webview + a single SQLite file open.
- **Reliability:** all imports run inside a SQLite transaction; a failed import must not leave partial `docs`/`versions`/`pages` rows.
- **Portability:** although V1 QA is Arch-only, avoid Arch-specific assumptions in the Rust/React code itself (e.g., don't hardcode `/usr/share` paths) so other distros and eventually other OSes are just a rebuild away.

---

## 12. Open Questions

1. `shiki` vs `prism` for syntax highlighting — `shiki` looks better but its grammar bundling affects final binary/app size; worth a quick spike before committing.
2. MDX in V1: full spec says "Markdown / MDX Renderer," but §7's security stance pushes hard toward treating MDX as "Markdown + a curated set of safe components" rather than arbitrary JSX. Confirm that's acceptable before promising full MDX component support in marketing copy.
3. Per-source `contentSelector` values (§6.3) need to be hand-mapped for each target doc site before the first scrape run — not blocking the app's architecture, but blocking having real content to test with.

> Resolved since the previous draft: bundle production is no longer an open question — §6 now defines the source-priority order, the TypeScript scraper architecture, and why a dedicated scraping framework (e.g. Scrapling) was evaluated and not adopted for V1.

---

## 13. Milestones (suggested)

1. **M1 — Core shell:** Tauri + React + Vite scaffold, SQLite schema migration on first launch, XDG path resolution.
2. **M2 — Import pipeline:** bundle import (§6) + MD/MDX → sanitized HTML compilation (§7).
3. **M3 — Reader UI:** sidebar tree, page rendering, code blocks + copy, dark mode.
4. **M4 — Search:** FTS5 schema + triggers, global search UI, scoped search UI.
5. **M5 — Arch packaging:** `.desktop` file, icons, AUR `PKGBUILD` (source + `-bin`), smoke-test under both X11 and Wayland.
