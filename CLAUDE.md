# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Homeroom is a single-file prototype web app (family courses, sessions, packages/payments, events, school calendars, circles, course groups). It was imported from the published Claude artifact `https://claude.ai/artifact/JsBnXnL4XwTc1vufFetLsL`. There is no build step, package manager, linter, or test suite. All HTML, CSS and JS (~2500 lines) live in one file.

- `index.html` and `homeroom.html` are byte-identical copies. Edit `index.html` and keep the other in sync (or delete it).
- Run it from a static server (e.g. `python -m http.server 8765` in this folder, then open http://localhost:8765/index.html). Fonts and supabase-js load from CDNs.
- To update the published artifact, republish the edited file with the Artifact tool using the URL above (read the live version first).

## Architecture (inside the `<script>` of index.html)

The code is layered top to bottom; views should only call lower layers:

1. **utils** — date helpers (dates are `YYYY-MM-DD` strings, times `HH:MM`), `esc()` for HTML escaping, `ic()` SVG icon set.
2. **store** — global `S` state persisted as JSON in `localStorage` key `homeroom.v2` via `persist()` / `loadState()` / `migrate()`. When adding fields to state, add defaults in `migrate()`. Real-family data (`S.mode==='real'`) is also mirrored to Supabase (project `Homeroom`, ref `pqganolgbvbybnshabsu`) by the **CLOUD SYNC** section: `persist()` calls `cloudQueue()`, which debounces `cloudSync()` to diff each collection against the last-synced snapshot and upsert/delete rows. `cloudLoad()` rebuilds `S` from the tables after sign-in. Demo data never syncs. Every table is `id` + `household_id` (or `household_ids[]` for circles/shared plans/course groups) + `data jsonb` holding the whole object, so adding a field to a record needs no migration; adding a new collection needs a table + RLS policy and an entry in `CMAP`/`CSHARED`. RLS (household membership; packages/payments need admin or finance-granted parent) is defined by the `homeroom_core_schema` migration; the client uses only the publishable key. Teacher sign-in and cross-household free/busy reads are not implemented server-side yet.
3. **business logic** — all session/money math. Key design rule: **package balances are derived, never stored as counters.** `ledger(pid,cid)` walks completed sessions in date order and pours deductible ones (`shouldDeduct`) into packages in number order, so a session can't be deducted twice. `forecast`/`cashflow` project renewals from it. Rescheduling (`doReschedule`) keeps the original session as history and creates a replacement that carries the entitlement. Conflict checks (`findConflicts`) cover course/event overlaps plus per-course travel buffers. Homework tasks are derived from completed sessions (`homeworkTasks`); status (pending/missing/done) is computed by `taskStatus`, not stored.
4. **permissions** — role model (admin, parent, member, teacher) driven by the current viewer `V`. Teachers see only courses with `teacherAccess` and visible students; members see only their own data and no finance.
5. **course groups / circles** — cross-household features. They must only expose free/busy or the group's shared space, never another household's calendar, courses, payments or progress.
6. **seed data**, then **UI state** (`ui`), **views** (`render()` and per-tab renderers; `TABS_ADMIN` defines navigation), `renderModal()` for sheets/forms, **actions** (mutating handlers that call `persist()` then `refresh()`), a **demo data upgrade** pass, **onboarding** (welcome screen: demo family vs. set up my family), and the **boot** IIFE.

Multi-household: `CH` is the current household id; most lookups (`activePeople`, `hhCourses`, `hhPayments`) are scoped to it.

## Gotchas

- Saved localStorage data wins over seed data, so seed changes don't reach existing browsers. The "demo data upgrade" section patches only records that exactly match the original seed; add fixes there.
- Recurring schedules materialize into sessions via `syncSessions()` over `S.settings.horizon` days (default 112). `purgeFuture` only removes untouched future sessions.
- Views build HTML strings; always pass user text through `esc()`.
- Theming uses CSS variables on `:root` with a dark-mode block and a `data-theme` override; keep both in sync when adding colors.
