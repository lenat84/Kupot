# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"ארבע הקופות" (Four Jars) — a Hebrew, RTL, kids' money app delivered as an **installable PWA**. One wallet receives money; the user divides it into four jars: **treats** (`treats`), **short-term savings** (`short`, 5% monthly interest), **long-term investment** (`long`, 10% monthly interest), and **giving** (`give`, parent matches each shekel). The long jar charges 25% tax on gains if withdrawn within a year — modeling real life.

Beyond the four jars, the child can also earn money via **home chores** (`chores`): a parent-managed list of named chores, each with a fixed reward and a daily/weekly reset. The child marks a chore done → it becomes **pending** → a parent **approves** it, which credits the reward **into the wallet** (so it flows through the normal divide-into-jars lesson). No PIN/gate — trust-based like the rest of the app.

The chores screen is a **day-by-day view**: it opens on today (real Hebrew date via `dayNavLabel()`), and the `‹ ›` nav (`choreDay` state + `choreDayShift()`) pages **backward** through earlier days as a read-only record (no future browsing; marking/approving/editing only on today). Daily chores list under the day; weekly chores get a separate "משימות השבוע" section. Each chore's completions are stored in a per-period `history` map (keyed by `dayKey()`/`weekKey()`); `choreStatusOn(chore, date, isToday)` derives available/pending/done/missed. `migrateChores()` backfills `history` from the old single `lastApproved` for existing/imported states.

## Architecture

There is **no build system, no framework, no dependencies, and no tests**. The entire application is a single file: `index.html` (~850 lines) containing inline CSS and inline vanilla JS. Web fonts load from the Google Fonts CDN; everything else is self-contained.

Supporting files: `sw.js` (service worker, offline cache), `manifest.webmanifest` (PWA install metadata), `icon-*.png` / `apple-touch-icon.png` (PWA icons).

### State model (the core of the app)

- A single global `state` object holds everything; `fresh()` defines its shape: `{name, wallet, jars:{treats,short,long,give}, principal:{short,long}, longStart, log:[...], lastInterest, lastBackup, chores:[...]}`. Each chore is `{id, name, emoji, amount, cadence:'daily'|'weekly', pendingAt, lastApproved, history:{[periodKey]:ts}}`; `history` (keyed by `dayKey()`/`weekKey()`) is the source of truth for per-period completion.
- Persisted via the `Store` object (`Store.load`/`Store.save`) to **`localStorage`** under key `arba-kupot-v1` (with an optional `window.storage` async layer tried first). This is browser-local — clearing browser data wipes it. Durable escape hatches (all in Settings, no server/cloud): the JSON file backup/restore, and **device transfer** — `stateCode()` base64-encodes the whole state into a `KUPOT1:`-prefixed copy/paste code, `decodeCode()` reverses it, and both file-import and code-restore funnel through the shared `applyImportedState()` (validates `jars`, normalizes, runs `migrateChores()`, saves).
- `state.log` is the source of truth for derived values. Money only enters jars by *dividing the wallet* — there is no free-add. Derived/calc helpers read the log rather than storing redundant totals: `incomeTotal()`, `earnedInterest(jar)`, `jarGains(jar)`, `earnedMatch(jar)`, `taxCalc(jar,W)`, `maxReceivable(jar)`, `longFreeDate()`/`isLongEarly()`. Prefer extending these over adding new stored fields.
- Migrations live inline in the boot IIFE at the bottom of the script — it backfills fields (`received`, `principal`, `longStart`) for older saved states. Add new optional fields the same way so existing users' data upgrades cleanly.

### UI structure

Four views — `home`, `detail`, `divide`, `chores` — are sibling `<div>`s toggled by `show()` driven by the `view` variable; each has a `renderX()` function. Tapping the wallet hero (or its 📒 היסטוריה pill) opens `openWalletHistory()` — a sheet (`#walletScrim`) listing the last 10 wallet money-ins (`jar==='wallet' && amount>0`: both manual income and `מטלה:` chore approvals). `refresh()` re-renders the active view. A right-side (RTL) slide-out **drawer** (`#navScrim`, opened by the `☰` `menuBtn`/`menuBtn2`) switches between the **קופות** section (`home`/`detail`/`divide`) and the **מטלות בית** section (`chores`). All user-facing strings are Hebrew; keep new strings Hebrew and the layout RTL.

## Running and deploying

- **Run locally:** serve the directory over HTTP (e.g. `python3 -m http.server`) — the service worker requires `http`/`https`, not `file://`. Then open the served URL.
- **Deploy:** push to `main`; the app is hosted via GitHub Pages from the repo root (`/`). Netlify drop also works since there's no build step.

## Critical gotcha: service-worker cache busting

`sw.js` is cache-first and caches `index.html`. After changing `index.html` (or any asset), **bump the `CACHE` constant in `sw.js`** (e.g. `kupot-v1` → `kupot-v2`). Otherwise already-installed users keep getting the old cached file and never see the change. The `activate` handler deletes caches whose name doesn't match the current `CACHE`, so renaming it is what triggers the refresh.
