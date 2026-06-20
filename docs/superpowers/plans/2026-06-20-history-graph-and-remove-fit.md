# History Graph + Remove Google Fit — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a 7/14/30-day water-intake bar graph (opened from a water-drop header icon) and remove the dead Google Fit integration — entirely within the single `index.html`.

**Architecture:** All data already exists in `state.logs` (logs are never pruned; only *display* is filtered to today). The only real logic is pure aggregation — extract it as `buildDaySeries(logs, range, dailyGoal, now)` and TDD it in isolation, then inline it into a new `StatsModal` component that reuses the existing `KeypadModal` shell and Tailwind theme. Removing Google Fit frees the header button the graph needs.

**Tech Stack:** React 18 + JSX (in-browser Babel), Tailwind v3 Play CDN — all pinned, no build step. Dev-only TDD uses Node's built-in test runner (`node --test`), nothing installed.

**Testing approach (important):** This repo intentionally has no test framework and cannot gain one without breaking the no-build-step constraint. Pure logic is unit-tested via `node --test` in the scratchpad (a dev aid, **not committed** — the verified function body is copied into `index.html`). UI changes are verified by loading the served page in a real browser (Preview/Chrome MCP): `#root` non-empty, console clean except the known Tailwind/Babel notices, and the behaviors listed per task.

**Commits:** The repo's rule is *commit only when the user asks*, and we're on `main` (so branch first). Task commit steps are written for completeness but are gated on user go-ahead in Task 6 — do not push to `main`.

**Scratchpad dir (dev-only test files):**
`/private/tmp/claude-501/-Users-galkerem-Documents-code-waterfall/42e3dcce-012b-4321-9528-c74ad4bccd0e/scratchpad`

---

## File Structure

- `index.html` — the entire app. All shipped changes land here:
  - Remove: GSI loader `<script>`, Fit config consts, `GoogleFitService`, Fit state/effect/toast/handler-block.
  - Add: `buildDaySeries` helper (near `generateId`), `StatsModal` component (after `KeypadModal`), `isStatsOpen` state + water-drop header button + `<StatsModal/>` mount in `App`.
- `CLAUDE.md` — doc sync (remove Fit sections, add the history-graph note).
- `<scratchpad>/daySeries.mjs` + `<scratchpad>/daySeries.test.mjs` — dev-only TDD of the pure function. Not committed.

---

## Task 0: Pre-flight

- [ ] **Step 1: Confirm Node has the built-in test runner**

Run: `node --version`
Expected: `v18.x` or higher (built-in `node:test` + `node --test`). If Node is missing/older, stop and report — the pure-logic TDD step depends on it.

- [ ] **Step 2: Confirm clean starting tree**

Run: `git -C /Users/galkerem/Documents/code/waterfall status --short`
Expected: only untracked `CLAUDE.md` (and this new `docs/` plan). `index.html` unmodified.

---

## Task 1: TDD the `buildDaySeries` aggregation (pure logic)

**Files:**
- Create (dev-only): `<scratchpad>/daySeries.mjs`
- Test (dev-only): `<scratchpad>/daySeries.test.mjs`

- [ ] **Step 1: Write the failing test**

Create `<scratchpad>/daySeries.test.mjs`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { buildDaySeries } from './daySeries.mjs';

// Local-time anchors so the function's setHours(0,0,0,0) math is unambiguous.
const NOON = new Date(2026, 5, 20, 12, 0, 0).getTime();           // 20 Jun 2026, local noon
const at = (daysAgo, hour = 9) => new Date(2026, 5, 20 - daysAgo, hour).getTime();
const sum = (days) => days.reduce((s, d) => s + d.total, 0);

test('empty logs -> zero series sized to range, scale floored by goal', () => {
  const r = buildDaySeries([], 7, 2000, NOON);
  assert.equal(r.days.length, 7);
  assert.ok(r.days.every(d => d.total === 0));
  assert.equal(r.maxScale, 2000);
  assert.equal(r.avg, 0);
  assert.equal(r.goalMetCount, 0);
});

test('buckets logs into the correct calendar day, oldest->newest', () => {
  const logs = [
    { amount: 500,  timestamp: at(0, 8) },
    { amount: 250,  timestamp: at(0, 20) },
    { amount: 1000, timestamp: at(2, 10) },
  ];
  const r = buildDaySeries(logs, 7, 2000, NOON);
  assert.equal(r.days[r.days.length - 1].total, 750);  // today is last
  assert.equal(r.days[r.days.length - 3].total, 1000); // 2 days ago
});

test('range trims to the last N days (inclusive of today)', () => {
  const logs = [{ amount: 999, timestamp: at(10) }];   // 10 days ago
  assert.equal(buildDaySeries(logs, 7, 2000, NOON).days.length, 7);
  assert.equal(sum(buildDaySeries(logs, 7, 2000, NOON).days), 0);    // outside 7-day window
  assert.equal(sum(buildDaySeries(logs, 14, 2000, NOON).days), 999); // inside 14-day window
});

test('maxScale follows the tallest day when it exceeds goal', () => {
  const logs = [{ amount: 3000, timestamp: at(0) }];
  assert.equal(buildDaySeries(logs, 7, 2000, NOON).maxScale, 3000);
});

test('avg (over full range) and goalMetCount', () => {
  const logs = [
    { amount: 2000, timestamp: at(0) }, // meets
    { amount: 2500, timestamp: at(1) }, // meets
    { amount: 1000, timestamp: at(2) }, // below
  ];
  const r = buildDaySeries(logs, 7, 2000, NOON);
  assert.equal(r.goalMetCount, 2);
  assert.equal(r.avg, Math.round((2000 + 2500 + 1000) / 7)); // 786
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `node --test /private/tmp/claude-501/-Users-galkerem-Documents-code-waterfall/42e3dcce-012b-4321-9528-c74ad4bccd0e/scratchpad/daySeries.test.mjs`
Expected: FAIL — cannot find module `./daySeries.mjs` (not created yet).

- [ ] **Step 3: Write the minimal implementation**

Create `<scratchpad>/daySeries.mjs`:

```js
export function buildDaySeries(logs, range, dailyGoal, now) {
    const todayStart = new Date(now);
    todayStart.setHours(0, 0, 0, 0);
    const days = [];
    for (let i = range - 1; i >= 0; i--) {
        const dayStart = new Date(todayStart);
        dayStart.setDate(todayStart.getDate() - i);
        const dayEnd = new Date(dayStart);
        dayEnd.setDate(dayStart.getDate() + 1);          // calendar day end (DST-safe)
        const start = dayStart.getTime(), end = dayEnd.getTime();
        const total = logs.reduce(
            (s, l) => (l.timestamp >= start && l.timestamp < end ? s + l.amount : s), 0);
        days.push({ ts: start, total });
    }
    const maxScale = Math.max(dailyGoal, ...days.map(d => d.total), 1);
    const avg = Math.round(days.reduce((s, d) => s + d.total, 0) / range);
    const goalMetCount = days.filter(d => d.total >= dailyGoal).length;
    return { days, maxScale, avg, goalMetCount };
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `node --test /private/tmp/claude-501/-Users-galkerem-Documents-code-waterfall/42e3dcce-012b-4321-9528-c74ad4bccd0e/scratchpad/daySeries.test.mjs`
Expected: PASS — `# pass 5`, `# fail 0`.

- [ ] **Step 5: No commit** (scratchpad files are dev-only and not part of the repo). The verified function body is inlined in Task 4.

---

## Task 2: Remove Google Fit from `index.html`

**Files:**
- Modify: `index.html` (`<head>`, config block, `GoogleFitService`, `App`)

- [ ] **Step 1: Remove the GSI loader script**

Delete from `<head>`:

```html
    <!-- Google Identity Services -->
    <script src="https://accounts.google.com/gsi/client" async defer></script>
```

- [ ] **Step 2: Remove the Fit config constants**

Delete these four lines from the `// --- CONFIGURATION ---` block (keep `DEFAULT_GOAL` and `STORAGE_KEY`):

```js
        const GOOGLE_CLIENT_ID = "310423440367-sku9mv7cr23hkn911gig8v56fcjumqj1.apps.googleusercontent.com"; 
        // New keys for saving the token
        const GOOGLE_TOKEN_KEY = 'waterfall_google_token';
        const GOOGLE_EXPIRY_KEY = 'waterfall_token_expiry';
        const GOOGLE_FIT_SCOPES = 'https://www.googleapis.com/auth/fitness.nutrition.write';
```

- [ ] **Step 3: Remove the entire `GoogleFitService` object**

Delete the whole block from the `// --- Google Fit Service ---` comment through its closing `};`, leaving `generateId` directly followed by a blank line and `// --- Components ---`. (Largest deletion — anchor on the unique comment markers `// --- Google Fit Service ---` and `// --- Components ---`.)

- [ ] **Step 4: Remove Fit state, simplify the mount effect, simplify `handleAddWater`, remove the toast**

In `App`, delete the Fit state:

```js
            // Google Fit State
            const [isFitConnected, setIsFitConnected] = useState(false);
            const [showFitSuccess, setShowFitSuccess] = useState(false);
```

Replace the mount effect:

```js
            useEffect(() => {
                const loaded = loadState();
                setState(loaded);
                
                // Initialize Google Fit
                GoogleFitService.init((connected) => {
                    setIsFitConnected(connected);
                });
            }, []);
```

with:

```js
            useEffect(() => {
                setState(loadState());
            }, []);
```

Replace `handleAddWater`:

```js
            const handleAddWater = async (amount) => {
                const timestamp = Date.now();
                const newLog = {
                    id: generateId(),
                    amount,
                    timestamp,
                    type: 'water',
                };
                setState(prev => ({
                    ...prev,
                    logs: [...prev.logs, newLog]
                }));

                // Log to Google Fit if connected
                if (isFitConnected) {
                    const success = await GoogleFitService.logWater(amount, timestamp);
                    if (success) {
                        setShowFitSuccess(true);
                        setTimeout(() => setShowFitSuccess(false), 2000);
                    } else {
                        // If log failed because token expired, update UI
                        if (!GoogleFitService.accessToken) {
                            setIsFitConnected(false);
                        }
                    }
                }
            };
```

with:

```js
            const handleAddWater = (amount) => {
                const newLog = {
                    id: generateId(),
                    amount,
                    timestamp: Date.now(),
                    type: 'water',
                };
                setState(prev => ({ ...prev, logs: [...prev.logs, newLog] }));
            };
```

Delete the success-toast JSX:

```jsx
                    {/* Google Fit Success Toast */}
                    {showFitSuccess && (
                        <div className="fixed top-6 left-1/2 transform -translate-x-1/2 z-50 bg-surface-light dark:bg-surface-dark shadow-xl rounded-full px-4 py-2 flex items-center gap-2 border border-slate-200 dark:border-slate-700 animate-slide-up">
                            <span className="w-2 h-2 rounded-full bg-green-500"></span>
                            <span className="text-xs font-bold text-slate-700 dark:text-slate-200">Synced to Google Fit</span>
                        </div>
                    )}
```

- [ ] **Step 5: Verify (browser)**

Run: `uv run python -m http.server 8000` (from repo root), load `http://localhost:8000/`.
Expected: page renders (drop counter, Quick Add, history); console shows only the Tailwind + Babel notices; **no** reference errors (`GoogleFitService`/`isFitConnected` undefined would blank the page); Network tab has **no** request to `accounts.google.com`. The header still has two buttons (the heart will be replaced in Task 3).

- [ ] **Step 6: Commit (gated — see Task 6)**

```bash
git add index.html
git commit -m "refactor: remove dead Google Fit integration"
```

---

## Task 3: Header water-drop button + `isStatsOpen` state

**Files:**
- Modify: `index.html` (`App`)

- [ ] **Step 1: Add the stats-modal open state**

Add alongside the other `useState` declarations in `App`:

```js
            const [isStatsOpen, setIsStatsOpen] = useState(false);
```

- [ ] **Step 2: Replace the heart button with a water-drop button**

Replace the first header button (formerly the Fit/`favorite` button) with:

```jsx
                                <button 
                                    onClick={() => setIsStatsOpen(true)}
                                    className="flex items-center justify-center w-12 h-12 rounded-full bg-surface-light dark:bg-surface-dark border border-slate-200 dark:border-slate-700/50 shadow-sm hover:scale-105 transition-transform text-slate-600 dark:text-slate-300"
                                    title="History"
                                >
                                    <span className="material-symbols-outlined">water_drop</span>
                                </button>
```

(The settings-gear button beside it is unchanged — out of scope.)

- [ ] **Step 3: Verify (browser)**

Reload `http://localhost:8000/`. Expected: the header shows a water-drop icon button (not a heart). Clicking it does nothing visible yet (modal added in Task 4) and throws no console error.

- [ ] **Step 4: Commit (gated — see Task 6)**

```bash
git add index.html
git commit -m "feat: water-drop header button opens history (state wired)"
```

---

## Task 4: `StatsModal` component + inline `buildDaySeries`

**Files:**
- Modify: `index.html` (add helper + component, mount in `App`)

- [ ] **Step 1: Inline the verified `buildDaySeries` helper**

Add directly after `generateId` (inside the babel script, top-level), copying the body validated in Task 1 (without `export`):

```js
        const buildDaySeries = (logs, range, dailyGoal, now) => {
            const todayStart = new Date(now);
            todayStart.setHours(0, 0, 0, 0);
            const days = [];
            for (let i = range - 1; i >= 0; i--) {
                const dayStart = new Date(todayStart);
                dayStart.setDate(todayStart.getDate() - i);
                const dayEnd = new Date(dayStart);
                dayEnd.setDate(dayStart.getDate() + 1);
                const start = dayStart.getTime(), end = dayEnd.getTime();
                const total = logs.reduce(
                    (s, l) => (l.timestamp >= start && l.timestamp < end ? s + l.amount : s), 0);
                days.push({ ts: start, total });
            }
            const maxScale = Math.max(dailyGoal, ...days.map(d => d.total), 1);
            const avg = Math.round(days.reduce((s, d) => s + d.total, 0) / range);
            const goalMetCount = days.filter(d => d.total >= dailyGoal).length;
            return { days, maxScale, avg, goalMetCount };
        };
```

- [ ] **Step 2: Add the `StatsModal` component**

Add after `KeypadModal` (before `// --- Main App Component ---`):

```jsx
        const STAT_RANGES = [7, 14, 30];

        const StatsModal = ({ isOpen, onClose, logs, dailyGoal }) => {
            const [range, setRange] = useState(7);
            if (!isOpen) return null;

            const { days, maxScale, avg, goalMetCount } = buildDaySeries(logs, range, dailyGoal, Date.now());
            const fmt = (ts, opts) => new Date(ts).toLocaleDateString('en-GB', opts);

            return (
                <div className="fixed inset-0 z-50 flex items-end sm:items-center justify-center bg-black/60 backdrop-blur-sm animate-fade-in">
                    <div className="w-full max-w-sm bg-surface-light dark:bg-surface-dark rounded-t-3xl sm:rounded-3xl p-6 shadow-2xl transform transition-transform animate-slide-up">
                        <div className="flex justify-between items-center mb-5">
                            <h2 className="text-xl font-bold text-slate-900 dark:text-white">History</h2>
                            <button onClick={onClose} className="p-2 rounded-full hover:bg-slate-100 dark:hover:bg-slate-700 transition-colors">
                                <span className="material-symbols-outlined text-slate-500">close</span>
                            </button>
                        </div>

                        <div className="flex gap-1 p-1 mb-6 rounded-xl bg-slate-100 dark:bg-slate-800">
                            {STAT_RANGES.map((r) => (
                                <button key={r} onClick={() => setRange(r)}
                                    className={`flex-1 py-2 rounded-lg text-sm font-bold transition-all ${range === r ? 'bg-primary text-white shadow-neon' : 'text-slate-500 dark:text-slate-400 hover:text-slate-700 dark:hover:text-slate-200'}`}>
                                    {r} Days
                                </button>
                            ))}
                        </div>

                        {logs.length === 0 ? (
                            <div className="text-center py-12 text-slate-400 text-sm italic">No water logged yet 💧</div>
                        ) : (
                            <>
                                <div className="relative h-40 mb-2">
                                    <div className="absolute left-0 right-0 border-t border-dashed border-primary/60 z-10" style={{ bottom: `${(dailyGoal / maxScale) * 100}%` }}>
                                        <span className="absolute -top-2 right-0 text-[9px] font-bold text-primary/80 bg-surface-light dark:bg-surface-dark px-1 leading-none">goal</span>
                                    </div>
                                    <div className="absolute inset-0 flex items-end gap-px">
                                        {days.map((d) => (
                                            <div key={d.ts} className="flex-1 flex items-end h-full" title={`${fmt(d.ts, { day: 'numeric', month: 'short' })} — ${d.total} ml`}>
                                                <div className={`w-full rounded-t transition-all duration-500 ${d.total >= dailyGoal ? 'bg-primary' : 'bg-primary/40'}`} style={{ height: `${(d.total / maxScale) * 100}%` }}></div>
                                            </div>
                                        ))}
                                    </div>
                                </div>

                                {range === 7 ? (
                                    <div className="flex gap-px mb-5">
                                        {days.map((d) => (
                                            <span key={d.ts} className="flex-1 text-center text-[10px] text-slate-400">
                                                {new Date(d.ts).toLocaleDateString('en-US', { weekday: 'narrow' })}
                                            </span>
                                        ))}
                                    </div>
                                ) : (
                                    <div className="flex justify-between mb-5 text-[10px] text-slate-400 font-medium">
                                        <span>{fmt(days[0].ts, { day: 'numeric', month: 'short' })}</span>
                                        <span>{fmt(days[days.length - 1].ts, { day: 'numeric', month: 'short' })}</span>
                                    </div>
                                )}

                                <div className="text-center text-xs text-slate-500 dark:text-slate-400">
                                    Avg <span className="font-bold text-slate-700 dark:text-slate-200">{avg.toLocaleString()} ml</span>/day
                                    <span className="mx-1.5 text-slate-300 dark:text-slate-600">·</span>
                                    goal met <span className="font-bold text-primary">{goalMetCount}/{range}</span> days
                                </div>
                            </>
                        )}
                    </div>
                </div>
            );
        };
```

- [ ] **Step 3: Mount `StatsModal` in `App`**

Add next to the existing modals (after the two `KeypadModal` mounts):

```jsx
                    <StatsModal isOpen={isStatsOpen} onClose={() => setIsStatsOpen(false)} logs={state.logs} dailyGoal={state.dailyGoal} />
```

- [ ] **Step 4: Verify (browser, with seeded multi-day data)**

With the server running, open `http://localhost:8000/`, then in the DevTools console seed ~30 days of varied logs and reload:

```js
const goal = 2000, logs = [];
for (let i = 0; i < 30; i++) {
  const day = new Date(); day.setHours(9,0,0,0); day.setDate(day.getDate() - i);
  const total = [0, 800, 1500, 2000, 2600, 3200][Math.floor(Math.random()*6)];
  for (let ml = 0; ml < total; ml += 500)
    logs.push({ id: 'seed'+i+'_'+ml, amount: 500, timestamp: day.getTime()+ml, type: 'water' });
}
localStorage.setItem('waterfall_data_v1', JSON.stringify({ dailyGoal: goal, logs }));
location.reload();
```

Expected after reload + clicking the water-drop button:
- Modal opens with the `7 / 14 / 30 Days` toggle; default 7 shows 7 bars with weekday-initial labels.
- Switching to 14 / 30 changes the bar count; 14/30 show a date-range caption instead of per-bar labels.
- Dashed "goal" line sits at goal height; bars ≥ goal are solid `primary`, below-goal bars are faded.
- Hovering a bar shows a `date — N ml` tooltip; the summary line shows a plausible avg and `goal met X/range`.
- Close button and tapping the backdrop-area button dismiss it; reopening preserves data.
- Screenshot each range via the Preview/Chrome MCP to confirm the look.

- [ ] **Step 5: Verify the live add-flow still updates the graph**

Clear the seed (`localStorage.removeItem('waterfall_data_v1'); location.reload()`), confirm empty-state ("No water logged yet 💧"), add water via Quick Add / keypad, reopen the modal → today's bar reflects the new total.

- [ ] **Step 6: Commit (gated — see Task 6)**

```bash
git add index.html
git commit -m "feat: 7/14/30-day intake history graph"
```

---

## Task 5: Update `CLAUDE.md`

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Sync the docs to the new reality**

- Remove the **Google Fit integration** section and the **Next task: fix Google Fit sync** section in full.
- Remove Google Identity Services / Google Fit mentions from the intro and **Architecture** bullets.
- Update the **Components** list: drop `GoogleFitService`; add `StatsModal` and note the `buildDaySeries` helper.
- Under **State & persistence**, add a sentence: full `logs` history now powers a 7/14/30-day graph in `StatsModal` (opened from the water-drop header button); only *today* is shown in `HistoryList`.

- [ ] **Step 2: Verify**

Re-read `CLAUDE.md`; confirm no remaining references to Google Fit, GSI, `GOOGLE_CLIENT_ID`, or the Fit "next task".

- [ ] **Step 3: Commit (gated — see Task 6)**

```bash
git add CLAUDE.md
git commit -m "docs: drop Google Fit, document history graph"
```

---

## Task 6: Full verification + commit handoff

- [ ] **Step 1: Final regression pass (browser)**

Hard-reload `http://localhost:8000/`. Confirm: clean console (only Tailwind/Babel notices), no `accounts.google.com` network call, Quick Add / custom keypad / edit-goal / delete-log all work, and the history graph behaves across 7/14/30 and the empty state.

- [ ] **Step 2: Commit (only if the user approves)**

The repo rule is *commit only when asked*, and we're on `main`. Confirm with the user, then branch and commit:

```bash
git -C /Users/galkerem/Documents/code/waterfall checkout -b feat/history-graph
git -C /Users/galkerem/Documents/code/waterfall add index.html CLAUDE.md
git -C /Users/galkerem/Documents/code/waterfall commit -m "feat: add 7/14/30-day history graph; remove dead Google Fit"
```

(If the user prefers, squash Tasks 2–5 into this single commit rather than per-task commits.)
