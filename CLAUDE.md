# Waterfall

A water-intake tracking PWA. The entire app is a **single self-contained `index.html`** —
all markup, styling config, and application logic live in that one file. Everything else in
the repo is dev/meta (`LICENSE`, `CLAUDE.md`, `docs/`, `.claude/`); only `index.html` ships.

Live: https://galke7.github.io/waterfall/

## Architecture

- **No build step.** No `package.json`, no bundler, no `node_modules`. You edit `index.html`
  and the change is the deploy.
- **React 18 + ReactDOM**, loaded as UMD bundles from the unpkg CDN.
- **JSX is transpiled in the browser** by Babel Standalone via
  `<script type="text/babel" data-presets="react">`. The whole app is inside that one
  script block.
- **Tailwind CSS** via the v3 Play CDN (`cdn.tailwindcss.com`), with the theme defined inline
  in a `tailwind.config = {...}` `<script>` in `<head>` (custom colors like `primary` #00ced1,
  `background-dark`, `surface-dark`, plus `fade-in` / `slide-up` animations).
- **Fonts/icons:** Google Fonts (Plus Jakarta Sans) + Material Symbols Outlined.

### Components (top → bottom in `index.html`)
`buildDaySeries` (pure helper: groups logs into per-day totals) · `CircularProgress` ·
`QuickAdd` · `HistoryList` · `KeypadModal` · `StatsModal` (the 7/14/30-day history graph) ·
`App` (root). Rendered with `ReactDOM.createRoot(#root)`.

## ⚠️ Critical constraints (learned the hard way)

- **Pin every CDN dependency to an exact version.** This app once went fully blank in
  production because the Babel script used the unpinned `latest` tag, which rolled from
  Babel 7 → 8. Babel 8 no longer applies the JSX preset implicitly, so the in-browser
  transpile failed and `#root` never rendered. Current pins: `@babel/standalone@7.29.7`,
  `react`/`react-dom@18.3.1`, `cdn.tailwindcss.com/3.4.17`. Do **not** revert any of these to
  floating tags.
- **Keep `data-presets="react"`** on the `text/babel` script — without it, JSX won't compile.
- **Failure mode is silent.** Because JSX compiles at runtime, a bad Babel/preset config
  produces a *blank page that still returns HTTP 200* — not a server error and not a failed
  Pages build. Always verify by loading the page and checking `#root` is non-empty + the
  console is clean.
- **No relative asset references.** Everything is absolute `https://` CDN URLs, so the app is
  independent of origin/subpath (works the same at `/waterfall/` as at `/`).

## State & persistence

- App state is `{ dailyGoal, logs: [...] }`, persisted to `localStorage` under
  **`waterfall_data_v1`**. Default goal is **2000 ml** (`DEFAULT_GOAL`).
- A log entry is `{ id, amount, timestamp, type }` (`type` is currently always `'water'`;
  `HistoryList` already has icon support for `tea`/`coffee`/`juice`). `id` from `generateId()`.
- "Today" is computed client-side from `timestamp >= startOfDay`. `HistoryList` shows only
  today's entries; the full `logs` history powers the **7/14/30-day** bar graph in `StatsModal`
  (opened from the water-drop button in the header). `StatsModal` aggregates via `buildDaySeries`.

## Deployment

- **GitHub Pages**, "legacy" build type, source = **`main` branch root**. Pushing to `main`
  auto-redeploys (a "pages build and deployment" Action run). The Pages *build API* `commit`
  field can lag/misreport the SHA — verify the deploy by fetching the live URL and checking
  `last-modified` + page content, not that field.

## Local dev / verify

It's a static file — serve the repo root and open `index.html`:

```bash
uv run python -m http.server 8000   # then open http://localhost:8000/
```

Expected-and-harmless console warnings: the `cdn.tailwindcss.com should not be used in
production` notice and the `in-browser Babel transformer` notice. Anything else (especially a
blank `#root`) means JSX failed to compile — check the Babel pin and `data-presets`.

