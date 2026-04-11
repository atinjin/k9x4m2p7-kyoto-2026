# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this repo is

A personal, static, single-page web app that plans a 4박 5일 (4-night / 5-day) Kyoto family trip for April 13–17, 2026. All user-facing text is in Korean. There is no backend, no build step, and no package manager — it is raw HTML/CSS/JS opened directly in a browser (or served statically).

The app has four pages rendered from the same file: trip overview, daily schedule, restaurant guide, and packing checklist.

## File layout

```
.
├── index.html            # Active SPA (≈1,374 lines). This is the file to edit.
├── kyoto_trip_2026.html  # Older/legacy version (≈403 lines). Kept for history; do NOT edit unless asked.
└── robots.txt            # Disallow all (private personal page).
```

There is intentionally no `package.json`, no `node_modules`, no bundler, no framework, and no test suite. Treat the repo as hand-written static files.

## Running / previewing

- Open `index.html` directly in a browser, or serve the directory with any static server (`python3 -m http.server`, `npx serve`, etc.).
- The schedule's map tab requires network access — it loads the Google Maps JS API from `maps.googleapis.com` with a hard-coded key at the bottom of `index.html` (see the `<script src="https://maps.googleapis.com/maps/api/js?...">` tag). If you touch that line, preserve the `callback=initMap&language=ko&region=JP` parameters.
- Fonts load from Google Fonts (`Noto Sans KR`).
- Offline: the page renders, but the map tab stays blank until the Maps script loads.

## Architecture of `index.html`

Everything lives in one file, organized as:

1. **`<style>` block** (lines ~8–474) — CSS custom properties in `:root` (`--bg`, `--pink`, `--blue`, `--green`, `--amber`, `--gray`, `--teal`, etc.) drive the entire color system. Reuse these variables instead of hard-coding hex values in new rules.
2. **`<body>` markup** (lines ~476–727) — static shells for each page:
   - `#page-overview` — fully static, hand-written HTML (hero, info cards, flights, stays, day summary, reservation status).
   - `#page-schedule` — empty containers filled in by JS (`#daySelectorWrap`, `#dayBanner`, `#schedSidebar`, `#schedMain`, `#timelineContent`, `#map-el`, `#map-waypoints`).
   - `#page-restaurant` — `#restGrid` filled by `renderRest()`.
   - `#page-packing` — `#packGrid` filled by `renderPacking()`.
3. **Main `<script>`** (lines ~729–1371) — data + vanilla-JS render functions (no framework, no build). Rendering is `innerHTML` templating with template literals.
4. **Google Maps loader `<script>`** (line ~1372) — must remain last so `window.initMap` is defined before the callback fires.

### Data model (top of the `<script>`)

All trip content is authored as plain JS object literals. When updating the trip, edit these constants — the UI re-renders from them automatically.

- `RESTAURANTS` (line ~731) — one object per restaurant. Fields: `id`, `name`, `nameEn`, `color`, `day` (e.g. `"1일차"`), `meal` (`"점심" | "저녁"`), `menu[]` (first entry is starred in UI), `tip`, `tel`, `reserve` (label), `rCls` (`r-ok` / `r-req` / `r-no`), `reserveHow`, `gmaps`. Consumed by `renderRest()` and by `DAYS[*].items[*].rid` cross-references via `RMAP`.
- `DAYS` (line ~742) — one object per day (1–5). Fields: `n`, `date`, `dow`, `title`, `color`, `wx {i,t,d}`, `outfit[]`, `gmaps` (full-day Directions URL), `items[]`, `meals[]`.
  - `items[]` entries: `{lat, lng, time, type, name, note, tags[], rid?}`. `type` ∈ `meal | sight | hotel | move` and is mapped to a color via `DCOL`. `tags[]` values are mapped to CSS classes via `TCLS` (`t-meal`, `t-sight`, `t-hotel`, `t-move`, `t-ok`, `t-req`). Keep tag strings consistent with `TCLS` keys or the tag will silently fall back to `t-move`.
  - `rid` links a schedule item to a `RESTAURANTS` entry so the name becomes a Google Maps link.
  - `meals[]` entries: `{tp, n, m, s, st, rid?, gm?}` where `s` ∈ `"ok" | "req" | ""` controls the reservation pill.
- `DCOL` / `TCLS` / `RMAP` — lookup tables. `RMAP` is built at load time from `RESTAURANTS`.
- `DAY_BANNERS` (line ~951) — per-day hero image + highlight chips. Index `n - 1` must line up with `DAYS`. Images are Unsplash hot-links with an `onerror` fallback.
- `PERSONS` (line ~1093) — four entries (`dad`, `mom`, `sis`, `me`) each with nested `bags[].items[]` for the packing checklist. Checklist state is in-memory only (`packState`, not persisted).

### Render / control flow

- `switchPage(p)` (line ~854) — toggles `.page.on`, syncs top nav pills, lazily calls `renderRest()` / `renderPacking()`, and resizes the Google map. Valid `p`: `overview | schedule | restaurant | packing`.
- `switchTab(t)` (line ~867) — switches schedule sub-pane: `sched | timeline | map`.
- `switchDay(n)` (line ~1070) → `renderDay(day)` (line ~1060) which calls `buildSidebar`, `buildBanner`, `buildTimeline`, and `renderMap`. Day selector buttons are generated on load at the bottom of the script.
- `initMap` (line ~878) is the Google Maps API callback. `renderMap(day)` clears/recreates markers + a dashed polyline path and builds the waypoint button strip under the map; waypoint buttons and marker click handlers sync each other's `.on` state.
- `renderPacking()` and `renderRest()` are idempotent via `if (g.children.length) return;` — they only render once. If you add a re-render path (e.g. filtering), remove or rework that guard.

There are no ES modules, no `import`/`export`, and no bundler — everything is in the global scope of that one `<script>` tag.

## Conventions to follow

- **Single-file philosophy.** Keep everything in `index.html`. Do not split into separate JS/CSS files, do not introduce a build tool, and do not add dependencies. If something can be solved with vanilla JS + a CSS variable, do it that way.
- **Language.** All UI strings, comments in markup, and data labels are Korean. Match the existing tone (polite, emoji-rich headings like `🌸`, `✈️`, `♨️`). Keep code identifiers and JS comments that already exist in Korean as Korean; do not translate them.
- **Colors.** Use the CSS custom properties from `:root` (`--pink`, `--blue`, `--green`, `--amber`, `--teal`, `--gray`, etc.). Day accent colors are mirrored in JS (`DAYS[*].color`, `DCOL`) — keep those in sync with the CSS palette.
- **Data-first edits.** For itinerary / restaurant / packing changes, edit the data arrays (`DAYS`, `RESTAURANTS`, `PERSONS`, `DAY_BANNERS`) — do not hand-edit rendered HTML, since it will be overwritten on re-render. The one exception is `#page-overview`, which is authored as static HTML.
- **Tag + status strings.** When adding schedule items, only use `tags` values that exist in `TCLS` (`조식`, `점심`, `저녁`, `포함`, `예약완료`, `예약불필요`, `예약필요`, `예약권장`, `관광`, `쇼핑`, `이동`, `휴식`, `숙소`). Otherwise the tag styling falls through to `t-move`.
- **Map coords.** `items[].lat/lng` are used for markers, the polyline, and `fitBounds`. Dummy/duplicate coordinates will cause overlapping pins — the renderer already deduplicates by `lat,lng` when building waypoint numbering.
- **`rid` consistency.** If you add a restaurant reference on a schedule item, make sure the `rid` matches an existing `RESTAURANTS[*].id`; otherwise the timeline link silently degrades to plain text.
- **HTML injection.** Data flows into the DOM through template-literal `innerHTML`. Content is authored by the repo owner, so escaping is not currently done — keep it that way unless you are adding user input, in which case you must escape.
- **No emojis in code you add** unless the user asks. The existing data already contains emojis because they are part of the UI copy — that is intentional and should be preserved when editing those strings.
- **Google Maps API key.** The key in the loader script is embedded in a public-facing file and is assumed to be restricted by HTTP referrer. Do not rotate, replace, or remove it without an explicit request. If the user asks about security, flag that it is publicly visible.

## Editing checklist

Before finishing a change to `index.html`:

1. The `DAYS[n-1]` index still matches `DAY_BANNERS[n-1]` (5 entries each).
2. Every `item.rid` / `meal.rid` points at a valid `RESTAURANTS[*].id`.
3. New `tags[]` entries exist as keys in `TCLS`.
4. Any new restaurant has a valid `rCls` (`r-ok` / `r-req` / `r-no`) — the CSS classes already exist.
5. New hex colors are either one of the existing CSS variables or added to `:root` (do not inline new brand colors).
6. If you add a new schedule subtab or page, update `switchPage` / `switchTab` and the corresponding nav-pill / `.ctab` markup.
7. Open the file in a browser and click through all four top-nav pages, all five day buttons, and the three schedule tabs (info / timeline / map). If you cannot run a browser, say so in the response instead of claiming success.

## Git workflow

- Default branch is `main`. Feature work happens on `claude/...` branches per the harness instructions in the task description.
- Commits in history are small and descriptive (e.g. `Add ryokan concierge reservation task and update waypoint buttons`). Match that style: a short imperative subject describing the user-visible change.
- Do not create PRs unless explicitly asked.
- Do not commit API keys or secrets beyond what is already tracked; flag it to the user if a request would add new secrets.

## Things NOT to do

- Do not introduce Node / npm / a bundler / TypeScript / a framework.
- Do not split `index.html` into multiple files.
- Do not silently delete or rewrite `kyoto_trip_2026.html`; it is the legacy prototype and is intentionally kept.
- Do not localize the UI to English.
- Do not add analytics, tracking, service workers, or external network calls beyond the existing Google Fonts / Google Maps script tags.
- Do not persist `packState` to `localStorage` unless the user explicitly asks — current behavior is session-only by design.
