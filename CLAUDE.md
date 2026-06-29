# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"ארבע הקופות" (Four Jars) — a Hebrew, RTL, kids' money app delivered as an **installable PWA**. One wallet receives money; the user divides it into four jars: **treats** (`treats`), **short-term savings** (`short`, 5% monthly interest), **long-term investment** (`long`, 10% monthly interest), and **giving** (`give`, parent matches each shekel). The long jar charges 25% tax on gains if withdrawn within a year — modeling real life.

## Architecture

There is **no build system, no framework, no dependencies, and no tests**. The entire application is a single file: `index.html` (~630 lines) containing inline CSS and inline vanilla JS. Web fonts load from the Google Fonts CDN; everything else is self-contained.

Supporting files: `sw.js` (service worker, offline cache), `icon-*.png` / `apple-touch-icon.png` (PWA icons). Note: `index.html` and `sw.js` both reference `manifest.webmanifest`, but that file is **not present in the repo** — recreate it if PWA install metadata needs to work.

### State model (the core of the app)

- A single global `state` object holds everything; `fresh()` defines its shape: `{name, wallet, jars:{treats,short,long,give}, principal:{short,long}, longStart, log:[...], lastInterest, lastBackup}`.
- Persisted via the `Store` object (`Store.load`/`Store.save`) to **`localStorage`** under key `arba-kupot-v1` (with an optional `window.storage` async layer tried first). This is browser-local — clearing browser data wipes it; the JSON backup/restore in Settings is the only durable escape hatch.
- `state.log` is the source of truth for derived values. Money only enters jars by *dividing the wallet* — there is no free-add. Derived/calc helpers read the log rather than storing redundant totals: `incomeTotal()`, `earnedInterest(jar)`, `jarGains(jar)`, `earnedMatch(jar)`, `taxCalc(jar,W)`, `maxReceivable(jar)`, `longFreeDate()`/`isLongEarly()`. Prefer extending these over adding new stored fields.
- Migrations live inline in the boot IIFE at the bottom of the script — it backfills fields (`received`, `principal`, `longStart`) for older saved states. Add new optional fields the same way so existing users' data upgrades cleanly.

### UI structure

Three views — `home`, `detail`, `divide` — are sibling `<div>`s toggled by `show()` driven by the `view` variable; each has a `renderX()` function. `refresh()` re-renders the active view. All user-facing strings are Hebrew; keep new strings Hebrew and the layout RTL.

## Running and deploying

- **Run locally:** serve the directory over HTTP (e.g. `python3 -m http.server`) — the service worker requires `http`/`https`, not `file://`. Then open the served URL.
- **Deploy:** push to `main`; the app is hosted via GitHub Pages from the repo root (`/`). Netlify drop also works since there's no build step.

## Critical gotcha: service-worker cache busting

`sw.js` is cache-first and caches `index.html`. After changing `index.html` (or any asset), **bump the `CACHE` constant in `sw.js`** (e.g. `kupot-v1` → `kupot-v2`). Otherwise already-installed users keep getting the old cached file and never see the change. The `activate` handler deletes caches whose name doesn't match the current `CACHE`, so renaming it is what triggers the refresh.
