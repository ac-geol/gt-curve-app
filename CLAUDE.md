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
per-block dimension columns (sub-blocked models), fixed parent cell, volume column, or a raw
mass column. All but the last multiply by density resolved in this precedence:
**density column (where populated) → per-domain override → global default**.

**Notices.** `notices` is rebuilt on every model build; `configNotices` is set once when
geometry is configured and survives rebuilds (currently only the sub-blocking warning, which
is re-seeded into `notices` only while `fixed` mode is active).

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
- **Domain values `0` and `false` are real domains** (waste is commonly coded 0) — filter on
  `null`/`undefined`/`''`, never on truthiness.
- **Column typing samples ~500 rows**, not row 0. A blank in the first row must not retype a
  grade column, and numeric-coded domains (`ROCKTYPE = 100/200`) must stay selectable as
  categories.
- CSV-derived text (column names, domain values) goes into the DOM via `new Option()` /
  `innerText`, never `innerHTML`.
- Contained metal is unit-aware via `#grade-unit`: g/t → troy oz (koz/Moz), % and ppm/ppb →
  tonnes of metal. Tonnage assumes coordinates in metres and density in t/m³.

## Verifying changes against the fixture

`example_block_model.csv` — 72,270 blocks, 20×20×10 m parent with 5×5×5 m sub-blocks,
`AU_GPT`/`CU_PCT`/`AG_GPT`, categoricals `ROCKTYPE`/`WEATHERING`/`CLASS`, per-block `DENSITY`,
and 547 deliberately unsampled blocks.

It should auto-detect into sub-block dimension mode and report, at zero cut-off:
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
