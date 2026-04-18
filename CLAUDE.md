# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

`index.html` is the entire application — a single-file, zero-build NPB (Nippon Professional Baseball) analytics tool written in Japanese. It visualises count-by-count batting average, RISP (runners-in-scoring-position) average, SLG, OPS, and a run-expectancy (RE) matrix for the 2025 Central League season. Chart.js is loaded via CDN; there are no other dependencies, no package manager, no test runner, and no lint config.

## Running / "building"

- Open `index.html` directly in a browser, or serve the directory (e.g. `python3 -m http.server 8000`) and navigate to it. There is no build step.
- Chart.js must be reachable on the network (loaded from `cdn.jsdelivr.net`).
- Any change is validated by reloading the page — there are no automated tests.

## Architecture

The whole app is three regions inside one HTML file:

1. **`<style>`** — dark-theme CSS; layout is a two-column grid that collapses to one column under 640px.
2. **Markup** — two tab panels (`#tab-operate`, `#tab-list`) sharing a header with team/year filters.
3. **`<script>`** — the data model, state, and render pipeline.

### Data layout (top of the script block)

All stats are hardcoded constants. When updating for a new season or adding teams, edit these in place:

- `AVG_DATA`, `RISP_AVG_DATA`, `SLG_DATA` — league-wide averages keyed by count string `"B-S"` (12 keys, `0-0` … `3-2`). Sourced from ProbSpace's per-PA final-result data (4,382 PAs).
- `AVG_SAMPLE`, `RISP_SAMPLE` — per-count AB counts, used for tooltips and the RISP-fallback threshold (<30 AB → fall back to normal avg; see `getRispAvg`).
- `BASE_AVG` — league AVG split by runner state, keyed by an 8-value base string (`---`, `1--`, `-2-`, `--3`, `12-`, `1-3`, `-23`, `123`). Used as a *multiplier* against `BASE_AVG['---']`, not as a standalone AVG.
- `RE_DATA` — run expectancy indexed as `RE_DATA[baseKey][outs]` (outs 0/1/2). Drives both the big number and the 8×3 matrix on the "一覧" tab.
- `TEAM_DATA` — per-team overrides (`T`, `G`, `D`, `S`, `C`, `DB`) each with their own `avg` and `runner` tables. `all` is intentionally `null` (meaning "use the league-wide defaults saved in `DEFAULT_*`").
- `TRIVIA` — ordered array of `{test, text}`; the first matching predicate wins and shows under the gauges.

`LEAGUE_AVG_NORMAL`/`LEAGUE_AVG_RISP`/`RISP_DIFF` exist so that `onTeamChange` can *estimate* team-level RISP by adding `RISP_DIFF` to the team's count AVG (per-team RISP data isn't collected yet — see the comment in `onTeamChange`).

### State and render pipeline

- `state` holds `{ bases:[1st,2nd,3rd], balls, strikes, outs }`. Every mutation (`toggleBase`, `addCount`, `resetAll`) ends with a single `updateUI()` call; do not render incrementally.
- `updateUI()` is the one place that reads state, computes derived stats, and writes the DOM. The key computation to preserve:
  - `baseAvgFactor = BASE_AVG[baseKey] / BASE_AVG['---']` — runner-state multiplier.
  - `isRISP = bases[1] || bases[2]` — picks `rispAvg` vs `normalAvg`.
  - `avg = min(0.999, baseCountAvg * baseAvgFactor)`; `obp ≈ avg * 1.1`; `ops = obp + slg` (clamped).
- Gauge maxes are hardcoded (`RE 3.00`, `AVG .500`, `SLG .800`, `OPS 1.200`) — changing max values means changing both the gauge math and the label text.
- The Chart.js instance is cached in `countChart`; `updateChart()` mutates `.data.datasets` and calls `update('none')` on subsequent renders. `onTeamChange` destroys and recreates it so the Y-axis rescales.

### Team filter behaviour

`onTeamChange(teamKey)` swaps `AVG_DATA` / `RISP_AVG_DATA` / `BASE_AVG` wholesale. For non-`all` teams it *derives* RISP by adding `RISP_DIFF` to each count AVG. The `DEFAULT_*` copies preserve the league-wide originals so switching back to `all` is lossless. After swapping, it rebuilds the count table and destroys the chart before calling `updateUI()`.

### Keyboard shortcuts

Handled globally in the `keydown` listener: `b`/`s`/`o` cycle balls (0→3), strikes (0→2), outs (0→2) modulo their max; `1`/`2`/`3` toggle bases; `r` resets. The handler early-returns inside form controls so the team `<select>` still works.

## Conventions

- **UI language is Japanese.** Labels, badges, trivia text, and tooltips are all Japanese — keep new strings consistent (e.g. `通常打率`, `得点圏打率`, `ノーアウト`).
- **Formatting.** Batting averages render via `fmtAvg` / inline `'.' + Math.round(v*1000).toString().padStart(3,'0')` (e.g. `.252`). RE values use `toFixed(3)`. Don't introduce a different format.
- **Colour semantics** are load-bearing: blue = RE, green = AVG, orange = SLG, pink = RISP, purple = OPS/trivia. Match existing CSS classes (`.val-re`, `.val-avg`, `.val-slg`, `.badge-risp`, etc.) rather than inlining new colours.
- **No frameworks.** Keep to vanilla JS, direct `document.getElementById` lookups, and string-template `innerHTML` for table rows — that matches the existing code.
- **Data provenance comments** (`// Source: ...`, `// ProbSpace ...`, `// nf3.sakura.ne.jp ...`) document where numbers came from; preserve them when editing the tables.
