# WC26 Predictor — Build-to-Product Design

**Date:** 2026-06-22
**Repo:** https://github.com/Avo-Sarkissian/World-Cup-Bracketing (currently empty)
**Goal:** Turn the Claude-designed UI prototype into a fully functioning, deployed product wired to real World Cup 2026 data.

## What we started with

A single self-contained Claude "Design Component" artifact (`World Cup Predictor.dc.html` + `support.js`).
`support.js` is the dc-runtime: it loads React/ReactDOM (UMD, CDN, pinned + SRI), parses the inline
`<x-dc>` template, and compiles `{{ }}` / `sc-for` / `sc-if` into a live React app. The logic
(`<script data-dc-script>`) is genuinely complete:

- 48 teams, 12 groups — the real 2026 draw.
- Group standings (predict: drag-to-order; live: sort by pts → GD → rating).
- Best-third-place ranking + FIFA's official Round-of-32 slotting (backtracking assignment that respects
  each slot's allowed group-letter options, winners never face their own third).
- Full knockout tree R32 → Final using the official FIFA match pairings (matches 73–104).
- Click-to-advance picks, golden-boot pick, Predict-vs-Reality compare, PNG share-card export, localStorage save.

**The one deliberate gap:** `fetchLiveData()` returns the embedded demo snapshot, with a comment inviting
"the Claude Code build" to swap in a real API returning `{ teams, fixtures, locked }`. Everything downstream
recomputes automatically.

## Decisions

### 1. Keep the dc-runtime (do not port)
The runtime renders the design with full fidelity (verified in a headless browser). Porting 790 lines of
custom-template logic to React/JSX is a large rewrite that risks design drift for zero user-visible gain.
Minimal, surgical path: keep `support.js` as a static asset, edit only the logic `<script>` to wire data.

### 2. Data source: openfootball/worldcup.json (free, keyless, CORS-enabled)
`https://raw.githubusercontent.com/openfootball/worldcup.json/master/2026/`
- `worldcup.teams.json` → `name` / `name_normalised` / `fifa_code` / `group`. The app's team ids are exactly
  the lowercased FIFA codes (`mex`=MEX, `cod`=COD, `irn`=IRN…), so the join is `fifa_code.toLowerCase() → id`.
- `worldcup.json` → all matches with `score.ft`, `group`, `date`, `time`. Group standings are computed from
  full-time scores (3/1/0 + goal difference).
- Served with `access-control-allow-origin: *` → the frontend fetches it **directly**. No backend, no key,
  no proxy. jsDelivr is used as a fallback mirror.
- Freshness: openfootball updates ~daily after matches (not in-play minutes). For a predictor whose core is
  standings → seeding, results-level data is authoritative and sufficient. In-play minute scores are a
  documented future enhancement behind the same seam.

### 3. Wiring (surgical edits to the logic script only)
- Keep the embedded snapshot as the offline/failure fallback (predict mode works with zero network).
- Rewrite `fetchLiveData()` to fetch + adapt openfootball into `{ teams:{id:{pts,gd}}, fixtures, locked, updatedAt }`.
- Add `applyLive(data)` to merge `pts`/`gd` into `TEAMS`, replace `FIXTURES` and `LOCKED`, then re-render.
- Add `componentDidMount()` to fetch on load; `onRefresh` re-fetches. Both fall back to the snapshot on error.
- `locked[group]` = 4 when all 6 group matches are FT, else 0 (honest; never shows a wrong "confirmed").
  Partial clinch detection is a future enhancement.
- Ticker: derive status from results (FT) and kickoff times (upcoming); show a window of recent + upcoming.
  Replace the hardcoded "Matchday 2" label with a derived feed label; surface an "offline → demo data" state.

### 4. Deployment: GitHub Pages (static, free, matches the repo)
Pure static site (`index.html` + `support.js` + `screens/`). A GitHub Actions workflow deploys `main` to
Pages. `.nojekyll` so dotfiles/underscores serve correctly. No secrets required.

## Success conditions (verifiable)
1. App loads at the Pages URL and renders all four views (Groups / Bracket / Your Picks / Compare).
2. On load it fetches openfootball and group standings reflect real results (verified: Group A etc. match the live feed).
3. With the network blocked, the app still renders fully from the embedded snapshot (no crash).
4. Drag-to-reorder, click-to-advance, golden boot, save, reset, surprise-me, share, download-card all work.
5. Repo is pushed to `Avo-Sarkissian/World-Cup-Bracketing` with a README and the Pages workflow.

## Out of scope (v1)
- In-play minute-by-minute scores (needs a keyed real-time API + proxy).
- Live knockout results overlay (the design intentionally shows TBD in live bracket until groups complete).
- Accounts / server-side persistence (localStorage is sufficient).
