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
paint, never `newPlot`.

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

**Units.** `#unit-system` (metric | imperial) is a *relabelling*, not a conversion:
ft × ft × ft × ton/ft³ is short tons exactly as m × m × m × t/m³ is tonnes, so the arithmetic
is identical and only `UNIT_SYSTEMS` labels change. `applyUnitLabels()` is the single writer
of every unit-bearing string (`.u-len/.u-vol/.u-mass/.u-dens` spans, the `bx/by/bz`
placeholders, the `ax-src-*` first option, chart axis titles) — do not hard-code a unit
anywhere else. Switching systems **resets** the density default and clears per-domain
overrides rather than reinterpreting a metric number under an imperial label.

The one real conversion is `T_PER_ST` in `fmtMetal()`: g/t is per tonne and opt is per short
ton, so contained metal converts when the grade unit disagrees with the tonnage unit. Ratio
units (%, ppm, ppb) are mass fractions and carry over untouched.

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
  ppm/ppb → tons of metal in whichever system is selected.
- **A tonnage factor is the reciprocal of a density**, and the imperial convention is volume
  per mass (~11.9 ft³/ton), not mass per volume (~0.084 ton/ft³). Reading one as the other is
  a ~140× silent error, and no column name distinguishes them — hence `#density-form` and the
  plausibility notice. Both forms are supported; neither is guessed.
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
