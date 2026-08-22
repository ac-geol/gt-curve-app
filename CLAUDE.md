# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, zero-build browser dashboard that turns a mining block model CSV into a
grade-tonnage curve. `dashboard.html` contains all markup, CSS and JS; PapaParse and Plotly
load from CDN. There is no package.json, no bundler, no test framework, and no git repo.
Editing the file and reloading the page is the entire development loop.

`example_block_model.csv` is a generated fixture with known totals — see *Verifying changes*.

## Running and testing it

`file://` is blocked by the Chrome automation extension, so serve over HTTP to test:

```bash
python3 -m http.server 8777          # then open http://localhost:8777/dashboard.html
```

There are no unit tests. Changes to the calculation path are verified by cross-checking the
app's own output against an independent computation over the same CSV:

```bash
node -e "…"   # re-derive tonnage/grade/metal straight from the CSV, compare to the metric cards
```

To drive the page without clicking, set a control's `.value` and dispatch the event the
listener actually binds (`change` for selects, `input` for range/number inputs) — the app has
no state outside the DOM plus the module-level `model`, so this is faithful to real use.
Loading a CSV programmatically requires a `DataTransfer` to populate `#csv-file.files`.

## Architecture

The JS is one IIFE-less script split into nine numbered sections. The parts that matter:

**Two-phase render — do not collapse these.** `rebuildAndRender()` re-reads every block and is
expensive (~2 s at 5M blocks). `renderCutoffOnly()` is O(log n) and runs on every slider frame
(~0.08 ms). The cut-off slider must only ever call the second one; it moves the marker via
`Plotly.relayout` and recomputes four numbers. The GT curve does not depend on the cut-off, so
re-plotting it while dragging is pure waste. `renderCurve()` uses `Plotly.react` after first
paint, never `newPlot`. The chart axis min/max boxes (`rx/ry/ry2-min|max`) are view-only for
the same reason: `applyChartRanges()` relayouts the existing plot, and a blank end is scaled
from `lastCurve` rather than left to Plotly, so half a range is still a usable instruction.

**The cumulative model** (`buildModel`) is the reason the slider is free. It filters, sorts
grades descending once, and builds `cumMass`/`cumMetal` as `Float64Array(n+1)` prefix sums.
`countAbove()` is an upper-bound binary search over the descending array; `queryCutoff()`
indexes the prefix sums. Anything that iterates all blocks per cut-off is a regression.

**Tonnage resolution** (`makeMassFn`) returns a closure chosen by the `#tonnage-mode` select:
block geometry (`geom`), a volume column, or a raw mass column. The first two multiply by
`makeDensityFn()`, which resolves in this precedence:
**density column (where populated) → per-domain override → global default**, and always
returns *mass per volume* — a tonnage factor is inverted once, at that single point.

**Per-axis geometry.** In `geom` mode each axis is sized independently by `makeExtentFn(ax)`,
driven by `#ax-src-{x,y,z}`: a fixed number (`b{x,y,z}`), a per-block size column
(`d{x,y,z}-col`), or a from/to extent pair (`lo-/hi-{x,y,z}-col`, subtracted as `|hi − lo|`).
Axes are free to differ — **fixed X/Y with a per-block Z is a normal model**, not a
misconfiguration, and was the reason the old all-or-nothing `dims`/`fixed` modes were merged.
Any axis returning `null` excludes the block; nothing is ever defaulted to 1.

**Grade units include `$/t`.** A value / NSR column is the usual cut-off variable on a
polymetallic model, and it is modelled exactly like a grade — the curve, the average and the
prefix sums are unchanged, only the labelling and `fmtMetal()` differ, where grade × mass is
already dollars and is scaled to thousand/million/billion. `gradeUnitSuffix()` is the single
reader of the grade unit label because `value` takes its suffix from `UNIT_SYSTEMS` ($/t vs
$/ton) rather than from `GRADE_UNITS`; `gradeNoun()` swaps Grade → Value in every label, axis
title and card.

**Units.** `#unit-system` (metric | imperial) is a *relabelling*, not a conversion:
ft × ft × ft × ton/ft³ is short tons exactly as m × m × m × t/m³ is tonnes, so the arithmetic
is identical and only `UNIT_SYSTEMS` labels change. `applyUnitLabels()` is the single writer
of every unit-bearing string (`.u-len/.u-vol/.u-mass/.u-massbig/.u-dens/.u-grade` spans, the
`.grade-noun/.metal-noun` wording, the `bx/by/bz` placeholders, the `ax-src-*` and `$/t`
option text, chart axis titles) — do not hard-code a unit anywhere else. It is called on a
grade-unit change too, since `$/t` moves with the system. Switching systems **resets** the density default and clears per-domain
overrides rather than reinterpreting a metric number under an imperial label.

The one real conversion is `T_PER_ST` in `fmtMetal()`: g/t is per tonne and opt is per short
ton, so contained metal converts when the grade unit disagrees with the tonnage unit. Ratio
units (%, ppm, ppb) are mass fractions and carry over untouched. `$/t` prices whichever ton
the tonnage is already in, so it converts nothing either.

**Notices.** `notices` is rebuilt on every model build. `unevenAxes` is set when geometry is
configured and survives rebuilds; `unevenFixedAxes()` narrows it to axes *still* on a fixed
size, which is the only case where uneven spacing misstates tonnage.
`densityPlausibilityNotice()` samples ~500 rows through `makeDensityFn()` and warns when the
median falls outside `DENSITY_RANGE` for the system. It never corrects the number — it says
what was used and names the likely cause.

## Domain rules that are easy to break silently

These caused real, wrong-but-plausible numbers in earlier versions. Regressions here do not
throw — they just produce a subtly wrong curve.

- **`null >= 0` is `true` in JS.** Blank grades must be rejected with an explicit
  `typeof v === 'number'` check (`num()`), or they enter the model as zero-grade dilution that
  inflates tonnage and depresses average grade. `undefined >= 0` is `false`, so blank and
  absent otherwise behave differently.
- **Never fabricate tonnage.** A missing mass, volume, dimension or density means the block is
  excluded and counted in a notice — never defaulted to 1, and never silently dropped.
- **Modal coordinate spacing is the *sub-block* size on a sub-blocked model, not the parent
  cell.** `inferBlockSize()` deliberately refuses to prefill the parent size when spacing
  coverage is `< 0.9`; prefilling it understated tonnage ~65× on the test model. If you touch
  the inference, keep the refusal.
- **The coverage test does not catch a variable-thickness axis.** Centroids of 1–10 m blocks
  still land on a fine regular grid, so Z reads back as a confident 0.5 m — a number that is
  not any block's size. `inferBlockSize()` therefore prefills `b{x,y,z}` **only for an axis
  whose source is already `fixed`**, and nothing re-runs inference when the user switches an
  axis onto `fixed` (a blank box excludes blocks loudly; a plausible wrong number does not).
  The hint says "centroid spacing" for the same reason — it is not a cell size.
- **Domain values `0` and `false` are real domains** (waste is commonly coded 0) — filter on
  `null`/`undefined`/`''`, never on truthiness.
- **Column typing samples ~500 rows**, not row 0. A blank in the first row must not retype a
  grade column, and numeric-coded domains (`ROCKTYPE = 100/200`) must stay selectable as
  categories.
- CSV-derived text (column names, domain values) goes into the DOM via `new Option()` /
  `innerText`, never `innerHTML`.
- Contained metal is unit-aware via `#grade-unit`: g/t and opt → troy oz (koz/Moz), % and
  ppm/ppb → tons of metal in whichever system is selected, `$/t` → dollars.
- **A tonnage factor is the reciprocal of a density**, and the imperial convention is volume
  per mass (~11.9 ft³/ton), not mass per volume (~0.084 ton/ft³). Reading one as the other is
  a ~140× silent error, and no column name distinguishes them — hence `#density-form` and the
  plausibility notice. Both forms are supported; neither is guessed.
- **A value / NSR column is only the *fallback* grade default**, and it has two tiers.
  `findValueCol()` runs after the element-grade regex, so a model carrying both `AU_GPT` and
  `NSR` still opens on the metal. `VALUE_COL_RE` (NSR / VALUE / VAL / dollar spellings) says
  what the column holds; `PLAN_COL_RE` (PLAN / STRAT) only says what it was run for, so a
  weak-tier name is accepted **only if the column is not also in `categoryCols`** — a numeric
  `STRAT` beside a `VALUE26` is a stratigraphy code, and charting it as dollars produced
  exactly the kind of confident nonsense this file exists to prevent. Both tiers allow the
  price-deck year suffix (`VALUE26`, `STRAT26`, `NSR_2026`) and stay anchored, so
  `VALIDATION`, `REVISION`, `INTERVAL`, `PLAN_ID` and `STRATIGRAPHY` do not match.
- **The grade-unit inference chain ends in an `else`.** Without it, loading a `$/t` model and
  then a `g/t` one leaves the unit on `$/t` and reports ounces as dollars. Every other default
  in `populateDropdowns()` is re-inferred per file; this one must be too.
- **The cut-off and average-axis range boxes are cleared when the grade column or unit
  changes** (`resetGradeAxisRanges()`), because a bound typed in $/t means nothing on a g/t
  curve — the cards stay right while the chart goes blank, which reads as a broken app. The
  tonnage boxes are kept: Mt is Mt whatever is plotted against it.
- **A grade column named `AU_OPT` must be auto-detected**, or `populateDropdowns()` falls
  through to `numericCols[0]` — a coordinate column — and plots confident nonsense. The
  imperial spellings (`_opt`, `_oz`) are in the detection regex for that reason.

## Verifying changes against the fixture

`example_block_model.csv` — 72,270 blocks, 20×20×10 m parent with 5×5×5 m sub-blocks,
`AU_GPT`/`CU_PCT`/`AG_GPT`, categoricals `ROCKTYPE`/`WEATHERING`/`CLASS`, per-block `DENSITY`,
and 547 deliberately unsampled blocks.

It should auto-detect into `geom` mode with all three axes on size columns (DX/DY/DZ) and
report, at zero cut-off:
**136.20 Mt @ 0.197 g/t Au = 862.5 koz**, with a notice excluding 547 blocks. Copper is leached
in the oxide zone, so `WEATHERING = OXIDE` vs `FRESH` on `CU_PCT` must give visibly different
curves.

## Performance envelope (measured, Chrome, 4.19 GB tab heap)

| Blocks | Load → first chart | Heap after load | Heap after edits | Full rebuild | Slider frame |
|---|---|---|---|---|---|
| 72k | 0.3 s | ~40 MB | — | 25 ms | 0.05 ms |
| 2M | 5.2 s | 675 MB | 1.25 GB | 0.6–0.8 s | 0.08 ms |
| 5M | 14.2 s | 2.0 GB | 2.5 GB | 1.5–2.0 s | 0.08 ms |

~5M blocks is the practical ceiling; 8–10M would likely OOM the tab. The dominant cost is
PapaParse's array-of-row-objects (~340 bytes/row for ~96 bytes of actual numbers), then the
pair-array allocation and sort in `buildModel`. Known unimplemented wins, roughly in order of
value: bin grades into ~10k bins and reverse-cumulative-sum them (removes the sort entirely),
columnar typed arrays fed by a streaming `chunk` parse (7–10× memory), debounced rebuilds (the
density-default input currently rebuilds on every keystroke), and moving aggregation to a Worker.

## Planned work (discussed, not started)

**Multi-model comparison** — the headline request: overlay a new block model against the
previous one to see how an update moved the curve. Scoped at ~350–400 lines, landing almost
entirely in §1 upload (a file *list*), §5 wiring (model rows, labels, colours) and §9 render
(N×2 traces, comparison table, per-model notices). §6–§8 barely move.

Do these first, in this order:

1. **`readConfig()` — snapshot the sidebar into a plain object once per rebuild**, and pass it
   to `makeMassFn(cfg)` / `makeDensityFn(cfg)` / `buildModel(rows, cfg)` instead of each
   reading `$('…').value` ad hoc. ~40 lines. This is what makes "every model was built the
   same way" true *by construction* rather than by discipline, and it makes the calculation
   path testable without a DOM.
2. **One shared config for all models, with explicit column mapping.** When a configured
   column is missing from a loaded model — a renamed `AU_GPT` between versions — refuse to
   guess and show a mapping row. A naive overlay attributes a *configuration* difference to
   the geology, which is this file's whole subject.
3. **Per-model notices.** "547 blocks excluded" is unambiguous with one model and useless with
   two; 12k blocks dropped for missing density otherwise reads as a tonnage decrease.
4. **Cross-model derived values:** the slider range (currently the 99th percentile of *the*
   model — take the max across models or a shifted curve gets clipped) and `gradeDp()`.

The deliverable is the **delta row**, more than the overlay: each model's tonnage/grade/metal
at the current cut-off, plus the difference against the baseline. Colour by model, dash by
series (solid tonnage, dashed grade), which reads to about three models. Cap at 3 — comparison
models keep their raw rows, so the block ceiling divides by N.

Rejected: concatenating models into one dataset with a synthetic domain column. It buys
nothing (N series are still needed to overlay) and makes per-model column mapping impossible.

**When to stop being one file.** Not at a line count — the single file *is* the distribution
story, and `<script type="module">` cannot load over `file://` (origin `null`), the same
restriction that forces `worker: location.protocol !== 'file:'` in the parse config. The real
trigger is vendoring the CDN libraries: that needs a build step anyway, and splitting the JS
then comes along for free. A dependency-free `node build.js` concatenating `src/*.js` plus the
vendored libraries into `dashboard.html` keeps the single-file artifact. Sizes measured
2026-08-21: PapaParse 19 KB, `plotly-basic` 0.95 MB (covers scatter, which is all this draws),
full Plotly 3.43 MB.
