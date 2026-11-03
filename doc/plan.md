# Sticky Notes — PWA first (Windows / Mac / iPhone)

> **This step only:** create `~/drvax-code-local/sticky-notes/` with `git init`, and lay out the empty directory tree from *Architecture* (plus `scripts/lib/`, `tests/{unit,api,e2e,fixtures,helpers,scripts}/`, `doc/`, `drizzle/migrations/`, `.github/workflows/`), using `.gitkeep` files for empty folders. Save this plan as `doc/plan.md`. **No code, configs or package.json yet.** Coding starts with milestone 0 once you say so.

## Context
A sticky-note app for Windows, Mac and iPhone. Notes must behave like **real sticky notes**: each one is its own small window that stays where you put it. They are not cards on a board inside one app window.

To ship without any app store, v1 is an **installable PWA**: Nuxt 3 on Cloudflare Workers, the same stack and standards as jewish-fun. A Tauri shell can come later (phase 2) and reuse the same app. It would add what a browser cannot do. Project: `~/drvax-code-local/sticky-notes`.

## What the PWA can and cannot do (be explicit with the user)
| Want | PWA v1 | Later (Tauri shell, phase 2) |
|---|---|---|
| Each note its own window, at a saved position | ✅ Installed PWA in Chrome/Edge: `window.open(..., 'popup,left,top,width,height')`. Multiple monitors through the Window Management API. | ✅ frameless |
| Always on top | ⚠️ One note at a time, through Document Picture-in-Picture (Chrome/Edge desktop) | ✅ any note |
| On the desktop, under other windows | ❌ | ✅ |
| No title bar | ❌ The title bar stays, but it is tinted the note's colour (`theme-color` per window) | ✅ |
| Global "new note" hotkey | ❌ Use the manifest `shortcuts` instead (right-click the Dock/taskbar icon → New note) | ✅ |
| Notes reopen at login | ⚠️ The user adds the app to Login Items / Startup once. On launch the app reopens every visible note. | ✅ |
| iPhone | ✅ Add to Home Screen, works offline, pile view | Home Screen widgets need a native shell |
| Mac Safari "Add to Dock" | ⚠️ Works, but has no Window Management or Picture-in-Picture. Chrome/Edge is the recommended install. | — |

## Product rules
**Desktop (installed PWA)**
- **Launcher window.** A small "Sticky Notes" window holds New note, the list of notes, Trash, Settings and Sign in. Opening the app reopens every note marked visible, each in its own popup window at its saved position. After that, the launcher can be minimised.
- **Note window** (`/note/[id]`):
  - The page is the note: coloured background and a thin toolbar for colour, pin on top, roll-up and trash.
  - **The note's text is Markdown.** The editor is TipTap with markdown shortcuts and live preview, like Typora:
    - Typing `**bold**`, `_italic_`, `~~strike~~`, `` `code` ``, `# ` (heading levels 1–3), `- `, `1. `, `- [ ] `, `> `, `---` or ```` ``` ```` formats the text as you type.
    - Pasted Markdown is turned into formatting.
    - Checklist boxes can be ticked.
    - Links open in the system browser.
  - **Source view:** a toolbar toggle (or Ctrl/⌘+/) shows the raw Markdown in a monospace textarea for editing directly, and switches back.
  - **Copy:** "Copy as Markdown" is in the toolbar menu.
  - The note saves automatically 300 ms after typing stops.
  - Six classic colours. The window's title-bar colour follows the note's colour.
- **Placement:**
  - Each window saves `screenX/screenY/outerWidth/outerHeight` on `resize`, and polls once a second for moves because browsers have no move event. A `screenId` from `getScreenDetails()` is saved too.
  - Placement is saved **per device** and never synced.
  - If a note's screen is missing, the note goes onto the primary screen and keeps its offset.
- **Pin on top** moves the note into a Document Picture-in-Picture window. Only one note can be pinned at a time, and pinning a second note unpins the first.

**iPhone (Home Screen PWA)**
- Notes appear as a loose overlapping pile of coloured squares, not a grid. Tap a note to open it full-screen. Swipe it away to send it to the trash. A + button makes a new note.
- Install help: the first time Safari opens the app it shows an "Add to Home Screen" coach mark. iOS can delete PWA storage that goes unused for a while, so signing in (sync) is what protects the notes.

## Architecture (jewish-fun / Backtick stack)
```
sticky-notes/
  pages/index.vue            launcher (desktop) / pile (phone, by viewport)
  pages/note/[id].vue        one note = one window
  pages/login.vue
  components/NoteEditor.vue, NoteToolbar.vue, NotePile.vue, NoteCard.vue
  composables/useApi.ts      apiFetch<T>() / useApiFetch (copy from jewish-fun)
  composables/useNotes.ts    Dexie (IndexedDB) store, the only thing the UI reads and writes
  composables/useSync.ts     WebSocket client, op queue, hlc
  composables/useNoteWindows.ts  open/restore/track popups, screen details, PiP
  utils/palette.ts           note colours as Tailwind tokens
  utils/hlc.ts, utils/merge.ts, utils/placement.ts   pure, unit-tested
  utils/markdown.ts          one config: TipTap extensions (StarterKit, TaskList/TaskItem,
                             Link, tiptap-markdown serializer), mdTitle(), mdToSafeHtml()
                             (markdown-it, html:false) for read-only previews in list/pile
  server/api/auth/*          Google OAuth, copied from Backtick (session cookie works: same origin)
  server/api/sync.get.ts     upgrades to the user's UserNotes Durable Object
  server/durable/UserNotes.ts   DO SQLite storage + WebSocket fan-out (hibernation API)
  drizzle/migrations/        D1: users, sessions (hand-authored SQL)
  public/manifest.webmanifest via @vite-pwa/nuxt (service worker, offline, shortcuts)
```
- **Rendering:** `ssr: false` for `/note/**` and `/`, so they work offline from the service worker. Only `/login` and the marketing page render on the server. Auth follows the Backtick pattern: non-global `middleware/auth.ts`, and `server: false` on every `useFetch`.
- **Local first:** every device keeps a full copy in IndexedDB. The note windows share it and tell each other about changes through a `BroadcastChannel('notes')`, so an edit shows up in the launcher list at once.

## Data model
- **`notes`** (synced): `id uuid, body_md text, color, created_at, updated_at, deleted_at, hlc`.
  - The body is **stored as Markdown text**, not editor JSON, so it stays readable everywhere: in the database, in exports, in a conflict copy, and later in a native iPhone widget.
  - The note title shown in the launcher list and the pile is the first line, stripped of Markdown.
- **`placements`** (IndexedDB only, this device): `note_id, screen_id, x, y, w, h, rolled_up, visible`.
- **Server:**
  - D1 holds `users` and `sessions`.
  - Each user's notes live in their own Durable Object's SQLite storage. That Durable Object is the only writer, so concurrent edits never race.

## Sync
- **Protocol:** on connect the client sends `since: <last hlc>` and receives every change after that. After that, each local change goes up as an op (`upsert` / `delete`) and the Durable Object sends it to the user's other sockets. While offline, ops queue in IndexedDB.
- **Conflicts:**
  - Colour and the trash flag use last-writer-wins by `hlc`.
  - If two devices both edit the body offline, the newer version wins and the older one is saved as a **"conflict copy"** note. No text is silently lost.

## Standards carried over from jewish-fun
- **`CLAUDE.md` + `AGENTS.md` symlink**, with the same §0 on brief responses and the code rules:
  - Strict TypeScript, no `any`.
  - Tailwind only. The palette lives in `utils/palette.ts`, which `tailwind.config.ts` imports.
  - Touch targets have `min-h-[44px] touch-manipulation`.
  - Every API call goes through `apiFetch`; an ESLint rule bans calling `$fetch`/`useFetch` directly.
  - Playwright finds elements by `data-test`.
  - `.codegraph/` index.
- **The Cloudflare rule, copied word for word:** never run anything that reaches Cloudflare without asking the user for that specific run. The user runs `deploy` themselves.
- **Mock gate:** `NUXT_AUTH_MOCK` (a fixed test user, no Google). It is refused outside dev/test by a fail-closed check, pinned by a unit test.
- **Staging:** `[env.staging]` with its own D1 and Durable Object namespace. `tests/unit/wrangler-env-isolation.test.ts` fails if staging binds a production resource.
- **Migrations:** hand-written `drizzle/migrations/NNNN_Name.sql` files, plus `scripts/db-migrate.mjs` with `--local` / `--remote` / `--check` (exit code 2 means a migration is pending). Timestamps are stored in Unix seconds.

## npm scripts (same conventions)
- **`npm run help`** pages through `scripts/lib/script-catalog.mjs`. Every script answers `-- --help` without doing anything:
  - node scripts call `exitOnHelp(import.meta.url)`, copied from `scripts/lib/cli-help.mjs`;
  - bash scripts call `exit_on_help`, from `sh-help.sh`.
- **No inline `&&` chains.** A chain goes in a script file. `tests/unit/cli-help.test.ts` enforces this and the `--help` rule.

| Script | Does |
|---|---|
| `dev` | `scripts/dev.sh`: Nuxt build + `wrangler dev --local` |
| `build`, `typecheck`, `lint`, `lint:fix` | as in jewish-fun |
| `test` | interactive menu |
| `test:unit` | Jest (`jest.unit.config.ts`): hlc, merge/conflict copy, placement fallback, **Markdown round-trip** (every supported syntax: md → editor → md unchanged), `mdTitle`, `mdToSafeHtml` strips `<script>`/raw HTML and `javascript:` links, cli-help, env isolation, mock gate |
| `test:serve` | `tests/scripts/serve.mjs`: one `wrangler dev --local` + D1 clone per worker (ports 8788+N, `serve-info.json`) |
| `test:api` | Jest `--runInBand` against `test:serve`: auth (mock), session, `/sync` WebSocket protocol (catch-up, fan-out to a second socket, offline replay, conflict copy) |
| `test:e2e`, `test:e2e:<area>` | Playwright run by area: `note`, `windows`, `pile`, `sync`, `offline` |
| `test:e2e:desktop` / `test:e2e:mobile` | Chromium desktop project / WebKit iPhone project |
| `test:smoke` | runs against a deployed site: `/api/health`, manifest + service worker served, mock sign-in on staging |
| `test:unlock` | clears the run lock |
| `db:generate`, `db:migrate[:remote/:staging]`, `db:backup[:remote]` | as in jewish-fun |
| `deploy`, `deploy:staging` | `scripts/deploy.sh <env>` behind the same deploy gate (clean tree, staging e2e passed on this commit) |
| `lockfile:check` | `npm ci` dry run |

## Testing
**Unit:** Jest over the pure utils and the server helpers.

**API:** the same model as jewish-fun `tests/api`.
- Tests run against the live `wrangler dev`.
- They reseed through `server/api/test/reset.post.ts`, an endpoint that exists only when `NUXT_TEST=1` and that also deletes the Durable Object's storage.
- `tests/helpers/api.ts` names a dead server clearly instead of failing every test.
- A run lock stops `test:api` and `test:e2e` from running at the same time.

**E2E (Playwright, real app, no fakes needed):**
- Specs import `test` from `tests/fixtures/base` (with `logProgress`), carry `@area` tags, and reseed in `beforeAll` through `reseed()`.
- `windows`: click New note and catch the popup (`context.waitForEvent('page')`). Type, resize, close and reopen the launcher, then check the note window comes back at the same size and position. Also check roll-up and that moving a note to the trash closes its window.
- `sync`: two browser contexts act as two devices. An edit in one appears in the other within 1 s. Take one context offline (`context.setOffline`), edit both, reconnect, and check for the conflict copy.
- `note` (markdown):
  - Type `**hi**`, `- [ ] buy milk` and `# Title`, and check that bold, a checkbox and a heading appear.
  - Tick the box, open Source view, and check the text reads `- [x] buy milk`.
  - Paste a Markdown block and check it is formatted.
  - Reload and check the Markdown is unchanged.
- `offline`: load once, go offline, reload, and check notes still open and can be edited.
- `pile`: WebKit iPhone 15 viewport. Create a note, swipe it to the trash, restore it from Trash.

**Manual checklist** (`doc/manual-checklist.md`) for what Playwright can't drive:
- the real install flow on Windows Edge, Mac Chrome and iPhone Safari;
- moving a note across two monitors;
- Picture-in-Picture pin on top.

**CI:**
- `ci.yml` runs on every push: `lockfile:check`, typecheck, lint, unit, api, e2e.
- `smoke.yml` runs every hour against production.
- `staging-e2e.yml` is the run the deploy gate accepts.

## Milestones
0. **Scaffold.** Nuxt + Worker + Durable Object stub, `CLAUDE.md`/`AGENTS.md`, script catalog + `--help` helpers, Jest/Playwright configs, CI.
   - *Verify:* `npm run help`, `test:unit` and `ci.yml` are green on an empty app.
1. **Local-only notes.** IndexedDB store, note windows, placement save/restore, launcher, editor, trash, phone pile, service worker + manifest.
   - *Spike first:* can the launcher reopen N note windows on launch? Chrome may allow only one popup per click. The likely fix is the Window Management permission, which lets one click open many popups. If that fails, the launcher offers "Reopen notes" (one click per note) as the fallback.
   - *Verify:* `test:e2e:desktop` (`windows`, `note`) and `test:e2e:mobile` (`pile`), plus the manual checklist.
2. **Auth + sync.** Backtick Google OAuth, UserNotes Durable Object, op queue, conflict copies.
   - *Verify:* `test:api` + `test:e2e -- sync offline`.
3. **Staging + deploy gate + smoke.**
   - *Verify:* the user runs `deploy:staging`, `test:smoke` passes against it, then the user runs `deploy`.
4. **(Phase 2, optional) Tauri shell.** It wraps the same SPA routes for frameless, always-on-top and on-desktop windows, a global hotkey and start at login. Later, iPhone widgets come through the native shell and the App Store.
