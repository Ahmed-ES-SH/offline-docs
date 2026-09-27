# PHASE 0 — Foundation & Hardening: Detailed Plan

**Parent:** [`MAIN_PLAN.md`](./MAIN_PLAN.md) → Phase 0
**Goal:** turn the pristine `create-tauri-app` scaffold into a sane, secured baseline before any feature work (Phases 1–6 assume everything here exists).
**PRD references:** §7.3 (capability lockdown), §7.4 (CSP), §3 (Tailwind), §8 (window/layout).
**Estimated scope:** 1–2 working sessions, no feature code.

---

## 1. Starting State (verified 2026-09-05)

| Area | State |
|---|---|
| Git | **Not a repo** — `git init` needed |
| Tailwind | `tailwindcss@4.3.3`, `postcss`, `autoprefixer` installed but **not wired** (no Vite plugin, no postcss config, no CSS import) |
| Plugins | `clipboard-manager`, `dialog`, `fs` crates in `Cargo.toml` + JS bindings in `package.json`, but **not registered** in `lib.rs` and **no permissions** in `capabilities/default.json` |
| CSP | `"csp": null` in `tauri.conf.json` |
| Rust | Only stock `greet` command; `tokio` (full) and `rusqlite` (bundled) declared, unused |
| Frontend | Stock demo `App.tsx`/`App.css`, no Tailwind classes, no router usage despite `react-router-dom` dep |
| Window | 800×600, title "offline-docs" |
| Other | `pnpm-workspace.yaml` has only `allowBuilds: esbuild`; no ESLint/Prettier/Vitest; README is template text |

---

## 2. Decisions to Lock In (before touching code)

1. **Drop `tauri-plugin-fs` entirely (recommended).** PRD §7.3 says expose nothing beyond what's needed. Import (Phase 2) works like this: the **dialog** plugin returns a folder/zip path → **Rust** reads the filesystem itself. The webview never needs fs access. Removing the plugin (from `Cargo.toml`, `package.json`, and capabilities) shrinks the attack surface and the binary. Re-add only if a concrete V1 need appears.
2. **Tailwind 4 wiring method: `@tailwindcss/vite` plugin** (the Tailwind 4 idiom), not the PostCSS plugin chain. `autoprefixer`/`postcss` deps can then be removed (Tailwind 4 handles vendor prefixing via Lightning CSS).
3. **Keep `react-router-dom`? Remove (recommended).** §3 explicitly says navigation is sidebar state, no router. Fewer deps = smaller bundle. Revisit only if deep-linking URLs in the webview is ever wanted.
4. **CSP strategy:** two configs — `app.security.csp` (production, maximal) and `app.security.devCsp` (adds Vite HMR allowances). Details in Task 5.
5. **Vitest: yes, adopt now** as the TS test runner (used from Phase 2 onward). Keep it in devDependencies with a trivial smoke test so the pipeline exists.

---

## 3. Task Breakdown (ordered)

### Task 1 — Git init + baseline commit
- `git init` (default branch `main`), verify existing `.gitignore` covers `node_modules/`, `dist/`, `src-tauri/target/`, `src-tauri/gen/schemas/`.
- Add `plan/` and the PRD to the commit — the docs ship with the repo.
- Commit message: `chore: scaffold baseline (Tauri 2 + React 19 + Vite 7)`.
- **Note:** `Cargo.lock` and `pnpm-lock.yaml` are committed (correct for an app, not a library).

### Task 2 — Wire Tailwind 4
- `pnpm add -D @tailwindcss/vite`
- `vite.config.ts`: add `tailwindcss()` to `plugins: [react(), tailwindcss()]`.
- Replace `src/App.css` with `src/index.css` containing only `@import "tailwindcss";`; import it in `src/main.tsx`.
- Delete `App.css`; remove `autoprefixer` + `postcss` from devDependencies (`pnpm remove autoprefixer postcss`).
- Smoke check: a `className="bg-red-500"` element renders red in `pnpm tauri dev`.

### Task 3 — Register plugins + lock down capabilities (§7.3)
- `src-tauri/src/lib.rs`: register the two kept plugins:
  ```rust
  .plugin(tauri_plugin_clipboard_manager::init())
  .plugin(tauri_plugin_dialog::init())
  ```
  (keep `tauri_plugin_opener` for now — "open in browser" for doc links is plausibly V1; **decision point during implementation**: if doc links will open in-app only, remove it too and strip `opener:default`.)
- Remove `tauri-plugin-fs` from `Cargo.toml` and `@tauri-apps/plugin-fs` from `package.json` (Decision 1).
- `src-tauri/capabilities/default.json` permissions become:
  ```json
  [
    "core:default",
    "opener:default",
    "dialog:allow-open",
    "clipboard-manager:allow-write-text"
  ]
  ```
  Nothing else — no shell, no fs scope, no broad dialog access (`dialog:allow-save` etc. added only when a feature needs them).
- Verify: from the devtools console, a denied command (e.g. `fs`) errors with a permission failure, while clipboard write works.

### Task 4 — Window & app identity (§8)
- `src-tauri/tauri.conf.json` window entry:
  - `"title": "Offline Docs"`, `"width": 1200`, `"height": 800`, `"minWidth": 900`, `"minHeight": 600`, `"center": true`.
- Leave `identifier: com.adev.offline-docs` as-is (renaming the product is a Phase 6 / pre-ship concern, PRD header note).

### Task 5 — Strict CSP (§7.4)
- Production `app.security.csp`:
  ```
  default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';
  style-src-attr 'unsafe-inline'; img-src 'self' data:;
  connect-src 'self' ipc: http://ipc.localhost;
  font-src 'self' data:; object-src 'none'; base-uri 'self'; form-action 'self'
  ```
  Rationale: `style-src-attr 'unsafe-inline'` is required by **shiki** (inline `style` attributes on highlight spans); `ipc: http://ipc.localhost` is required by Tauri v2's IPC; everything external is blocked — the app is offline at render time (§7.4 "CSP can be aggressive").
- Dev `app.security.devCsp`: same, plus `connect-src 'self' ipc: http://ipc.localhost ws://localhost:1421 http://localhost:1420` for HMR.
- **Verify both ways:** `pnpm tauri dev` still hot-reloads; a hand-injected `<script>alert(1)</script>` in rendered HTML is blocked by CSP (check devtools console reports the CSP violation).

### Task 6 — Remove template cruft
- Rust: delete the `greet` command + `generate_handler![greet]` (empty handler list is fine until Phase 1 — or keep a temporary `ping` returning `Ok(())` so the invoke path is exercisable).
- Frontend: gut demo `App.tsx` → minimal three-region skeleton (header / aside / main as plain Tailwind divs, no logic), delete `src/assets` React/Vite logo imports, remove `public/tauri.svg` + `public/vite.svg`, update `index.html` (title "Offline Docs", drop vite favicon).
- README: rewrite briefly — project name, one-paragraph purpose (offline docs reader per PRD), prerequisites (rust, nodejs, pnpm — per §2 build deps), `pnpm install` / `pnpm tauri dev` quickstart, pointer to `plan/MAIN_PLAN.md` and the PRD.

### Task 7 — Tooling baseline
- ESLint 9 flat config + `typescript-eslint` + `eslint-plugin-react-hooks` + `eslint-plugin-react-refresh`; Prettier (`.prettierrc` minimal, print width 100).
- `package.json` scripts: `"lint": "eslint src"`, `"format": "prettier --write src"`, `"test": "vitest run"`, plus keep `tauri`/`dev`/`build`.
- Vitest + `@testing-library/react` + `jsdom` in devDependencies; one smoke test (`src/App.test.tsx`: renders the three regions) to prove the pipeline.
- Rust: `cargo fmt` + `cargo clippy -- -D warnings` already runnable; add a `justfile` or npm scripts alias (`pnpm rust:check`) — optional, pick one and document in README.
- **Out of scope here (explicitly deferred):** CI workflows (no remote yet), husky hooks. Revisit at Phase 6 release engineering.

---

## 4. Final State Checklist (acceptance)

- [ ] `git` repo with baseline commit; working tree clean after all tasks.
- [ ] `pnpm tauri dev` opens a 1200×800 "Offline Docs" window; Tailwind classes render.
- [ ] From the webview: clipboard write succeeds; a denied capability (fs) fails with a permission error.
- [ ] CSP: injected inline `<script>` is blocked; HMR still works in dev.
- [ ] `pnpm lint`, `pnpm test`, `tsc --noEmit`, `cargo clippy -- -D warnings`, `cargo fmt --check` all pass.
- [ ] `grep -r "plugin-fs"` empty; `greet` gone; no template SVGs/CSS remain.
- [ ] README reflects the actual project.

## 5. Handoff to Phase 1

Phase 0 leaves behind: a clean git history, a Tailwind-styled three-region shell with no logic, registered dialog/clipboard plugins with minimal capabilities, an enforced CSP, and lint/format/test tooling ready to be filled by the Phase 1 schema + command layer. **Nothing in Phase 0 touches the DB, migrations, or XDG paths** — that's Phase 1's first task.
