# OneSheet

A working spreadsheet — real formula engine, real `.xlsx` import — in **one HTML file** that runs
offline in a modern browser.

Open `index.html`. That's it. No build step, no bundler, no framework, no server.

```
5 000 rows × 702 columns (A–ZZ)   ·   ~45 functions   ·   260 KB unminified   ·   0 dependencies*
```

<sub>\* except SheetJS, fetched from cdnjs the first time you touch `.xlsx` — and never before.</sub>

---

## What's in it

| | |
|---|---|
| **Formula engine** | Pratt/precedence-climbing parser, dependency DAG, incremental recalc via Kahn topological sort, cycle detection. Never a full-sheet recalc. |
| **Formula X-Ray** (`Ctrl+Shift+X`) | Animated SVG dependency graph over the grid. Precedents flow in blue, dependents out in orange, depth by opacity. Edit the cell while it's open and the ripple animates through the dependents in true topological order. |
| **Time Travel** (`Ctrl+Shift+T`) | Scrub the entire edit history on a slider, play it back at ×1/×4/×16, restore any point. Undo/redo is cursor movement over the same log — one system, two features. |
| **Excel 97 mode** (`Ctrl+Shift+9`) | Full retro reskin: silver chrome, beveled everything, a working menu bar, a fake loading dialog, and Clippy. Every feature still works while skinned. |
| **Live widgets** | `=SLIDER(0,100,5,42)`, `=CHECKBOX()`, `=RATING(5)`. A widget *is* its formula, so it survives copy/paste, undo, time travel and the URL hash for free. A whole drag collapses into one undo step. |
| **Goal Seek** | Bracket expansion → bisection, secant fallback. Animates ~30 sampled iterations at 20 fps before landing. |
| **Charts & sparklines** | Hand-rolled SVG, no chart library. `F9` auto-detects bar/line/pie/scatter from the shape of your selection. Charts are floating, draggable and live. |
| **`=AI()`** | Mock Mode by default: deterministic, offline, zero-config — `hash(prompt+input)` seeds the answer, so a demo recorded today looks identical tomorrow. Add an Anthropic key in Settings and the same formula calls `claude-haiku-latest`, resolving asynchronously through the normal recalc path. |
| **IO** | CSV and real `.xlsx` both ways, drag-and-drop, autosave to localStorage, and share links that deflate the whole sheet into a URL hash. |

Everything is reachable from the command palette (`Ctrl/Cmd+K`).

## Architecture

Five modules, strict one-way dependencies. `ENGINE` contains no DOM references at all.

```
┌────────────┐   ┌────────────┐   ┌────────────┐
│   ENGINE   │◄──│  COMMANDS  │◄──│     UI     │
│ cells,deps │   │ undo/redo, │   │ grid,panels│
│ parse,calc │   │ history log│   │ palette    │
└────────────┘   └────────────┘   └────────────┘
       ▲                                ▲
       │        ┌────────────┐          │        ┌────────────┐
       └────────│     IO     │──────────┘        │  FEATURES  │
                │ csv, xlsx, │                   │ xray, time-│
                │ persistence│                   │ travel, ai │
                └────────────┘                   └────────────┘
```

Two decisions carry most of the weight:

**Every mutation goes through one command object.** That single choke point is what makes undo,
redo, time travel and autosave the same feature. History entries store *exact* cell specs rather
than deltas, so replaying in either direction is idempotent — which is why a scrub to any point in
a 5 000-event log lands in well under a frame.

**Ranges are watched by rectangle, not by cell.** `=SUM(A1:A5000)` costs one watcher in a
row-bucketed index instead of 5 000 graph edges, and ranges evaluate lazily over the sparse map —
700 000 empty cells are never materialised.

## Poke at it

The engine is drivable from the console; open devtools and reward yourself.

```js
OneSheet.engine.set("A1", "=SUM(B1:B10)")
OneSheet.engine.value("A1")
OneSheet.commands.undo()
OneSheet.time.seek(30)
OneSheet.selfTest()          // or load index.html?test=1
```

`?test=1` runs the §14 acceptance checklist against the live engine — parser, coercion, all ~45
functions, fill patterns, reference rewriting, cycles, time-travel seeks, Goal Seek convergence,
mock-AI determinism — and prints a pass/fail table to the console. It restores your sheet afterwards.

## Deliberate deviations from the spec

Four places where following the letter and the spirit pulled apart. Each is commented in the source.

1. **`-2^2` evaluates to `-4`, not `4`.** The §5.1 grammar puts `unary` above `power`, so
   `-2^2` parses as `-(2^2)`. Excel does the opposite. An explicit formal grammar beats a general
   "keep it Excel-ish" note, so the grammar won.
2. **"Restore here" keeps the log instead of truncating it.** Restoring applies one new command
   that transforms present → past. The history stays intact, so the scrubber still shows where you
   came from and `Ctrl+Z` walks back out — which is what "this itself is one undoable command"
   needs in order to mean anything.
3. **Reverse dependencies live in a global index, not on the cell.** §4 puts `rdeps` on `Cell`, but
   an *empty* cell can be referenced without existing in the sparse map. Precedents (`refs`,
   `ranges`) stay on the cell as specified.
4. **Imported formulas are downgraded when they use functions we don't implement.** §8.7 says keep
   formulas "when they parse". `SUMPRODUCT(...)` parses fine and would then evaluate to `#NAME?`,
   replacing a number Excel had already computed. Keeping the value and tagging the cell is the
   honest reading of "the toast is honest about what was kept".

## Constraints honoured

- One file. HTML, CSS, JS, icons (inline SVG), sounds (WebAudio) all live in `index.html`.
- Vanilla ES2021+. No React, Vue, jQuery, or chart library.
- Not minified — readable source is part of the product. 260 KB against a 500 KB budget.
- Kill the network entirely and everything works except `.xlsx` and real-mode AI, both of which
  degrade with an explanation rather than an error.

---

*Planned by Claude Fable 5. Built by Claude Opus 5. Human code: zero.*
