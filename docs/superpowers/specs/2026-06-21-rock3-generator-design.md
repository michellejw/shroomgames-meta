# Mycogrid Generator — Design Spec

**Date:** 2026-06-21
**Big Rock:** #3 (Solver + generator + daily/archive) — Phase 2 (Generator)
**Status:** Design approved, ready for implementation plan

## Context

Big Rock #3 Phase 1 (the uniqueness-validator solver) is complete and merged
(PR #1). It ships as the `MycogridSolver` SwiftPM package under
`rootline/scripts/`, with two CLI targets so far: the solver library and
`mycogrid-validate` (pool audit + single-file validation).

This spec covers **Phase 2 — the generator**: an offline authoring tool that
produces a large, validated pool of puzzles per tier so the hand-curated pool
in `PuzzleData.swift` can eventually be replaced. The generator is the
prerequisite for Phase 3 (daily/archive UI), which maps calendar dates into
this pool.

Terminology note: the genre is never called "Slitherlink" in code or docs. The
product is **Mycogrid**; the repo is **rootline**.

## Goals

- A new `mycogrid-generate` CLI that emits a validated JSON bundle of puzzles,
  grouped by tier.
- Every emitted puzzle is **uniquely solvable by pure logic** — the solver
  cracks it with constraint propagation alone, zero guessing.
- Reproducible output: same seed + same code → byte-identical bundle.
- Reuses the existing `Solver` / `PuzzleModel` / `Edge` / `Cell` types — one
  source of truth for the rules.

## Non-goals (explicitly out of scope)

- **Wiring the app to read the bundle.** Replacing `PuzzleData.swift`'s
  hand-curated lists with bundle loading is Phase 3. This spec stops at
  "validated bundle exists on disk."
- **Date → puzzle mapping, daily view, archive view.** All Phase 3.
- **Aesthetic shape curation.** Any valid uniquely-solvable region is fine
  (decision below); the generator does not try to make regions look like
  recognizable silhouettes.
- **Enriching the solver with more deduction rules.** The generator works with
  the solver's current two propagation rules (clue-count + dot-degree). See
  "Density emerges" below.

## Key decisions

These were settled during brainstorming and are the load-bearing constraints
for the design:

1. **Tier = grid size (unchanged).** Tiers stay fixed sizes — Sprout 4×6,
   Mycelium 5×7, Ancient 6×9, Old Growth 7×10 — matching the shipped app. The
   generator does not introduce an independent difficulty axis; it produces
   uniquely-solvable puzzles at each fixed size.

2. **Fixed committed bundle, ~100–200 puzzles per tier.** The generator is an
   offline authoring tool run by hand. Its output is a JSON file committed as a
   future app resource. Re-run with a fresh seed range to extend runway later.

3. **Pure-logic only (`guesses == 0`).** A puzzle is admitted only if the
   solver returns `.unique` *and* `trace.guesses == 0`. No backtracking. This
   guarantees every puzzle is reasoned step-by-step by a human, never guessed —
   aligned with Rootline's calm/meditative north star.

4. **Density emerges (greedy hiding to the pure-logic cap).** The solver
   currently knows only two propagation rules, which caps how many clues can be
   hidden while staying guess-free — especially on the larger boards. Rather
   than fight this, the generator hides as many clues as the pure-logic bar
   allows per board, and accepts whatever shown-clue density results. The
   "most clues → minimal clues" gradient emerges from grid size. Larger tiers
   will likely show more clues than today's hand-tuned sets; that is acceptable
   for v1 and revisited (by teaching the solver more rules) only after looking
   at real generated boards.

5. **Any valid region is fine.** No shape-quality heuristics. The generator
   needs simply-connected, uniquely-solvable regions; the visual silhouette is
   incidental. The "grove" framing lives in the tier names, not the shapes.

## Architecture

A new CLI target, `mycogrid-generate`, added to the existing `MycogridSolver`
SwiftPM package — a sibling to `mycogrid-validate`. It reuses `Solver`,
`PuzzleModel`, `Edge`, and `Cell` so there is no duplicated rule logic.

### Per-puzzle generation pipeline

1. **Generate a region.** Grow a random simply-connected `inside` region of the
   tier's grid size (see "Region generation" below).
2. **Derive the full puzzle.** `PuzzleModel` produces the solution loop and
   every cell's true clue value.
3. **Gate on the fully-clued board.** Run `solve` with *all* clues shown.
   Require `verdict == .unique` and `trace.guesses == 0`. A region whose fully-
   clued board already needs guessing can never become pure-logic once clues
   are hidden — discard it and generate another region.
4. **Hide clues greedily.** Repeatedly pick a random still-shown clue, remove
   it, and re-run `solve`. Keep the removal only if the puzzle stays `.unique`
   with `guesses == 0`; otherwise restore the clue and mark it
   "cannot-hide". Stop when every remaining shown clue has been tried and
   none can be removed. The final shown-clue count is whatever the pure-logic
   cap allows.
5. **Record & emit.** Store the puzzle (`inside`, `hideClues`) plus its
   measured signal (`shownClueCount`, `rulesFired`, `seed`).

### Driver

The driver loops the pipeline per tier until it has the requested count,
deduping identical regions (by region identity — see "Identity" below), then
writes a single JSON bundle covering all tiers (or one tier if `--tier` is
given).

## Region generation

**Approach: cell accretion.** Start from a random seed cell, then repeatedly
add a random edge-adjacent cell. Two guards:

- **No holes.** An inside region with a hole produces *two* loops, not one.
  After building the region, flood-fill the *outside* cells from the grid
  border; if the outside is not a single connected component, the region has a
  hole — discard.
- **Non-trivial fill.** Stop accretion at a target fill fraction (~40–60% of
  the grid) so the region isn't so large that its loop is just the grid border.

This maps directly onto the existing `inside: Set<Cell>` representation and is
easy to test. (Alternative considered: generating a random simple cycle on the
dot lattice directly. Rejected — the random-cycle algorithms are fiddlier and
the no-hole / non-trivial properties are harder to control. Since any valid
region is acceptable and the solver gate does the real quality filtering,
accretion's simplicity wins.)

## Determinism

The generator takes a `--seed <int>` and drives all randomness from one seeded
PRNG — an explicit reproducible generator (e.g. SplitMix64), **not** Swift's
`SystemRandomNumberGenerator` (which is not reproducible across runs). Same
seed + same code → byte-identical bundle.

This gives reproducible, auditable output (a reviewer can regenerate and
`git diff` to confirm the committed bundle wasn't hand-edited — the same drift-
guard pattern as the tokens `check:tokens` step) and controllable runway
extension (bump to a new seed range, keep existing puzzles stable).

### CLI shape

```
mycogrid-generate --seed 1 --count 150 --out bundle.json
mycogrid-generate --seed 1 --count 150 --tier mycelium --out mycelium.json
```

- `--seed <int>` — required; the PRNG seed.
- `--count <int>` — puzzles per tier.
- `--tier <name>` — optional; restrict to one tier (default: all four).
- `--out <path>` — output file path.

## Output format

One JSON file, grouped by tier. Each puzzle entry:

- **`id`** — a stable, content-derived identifier (short hash of
  `cols,rows,inside`). Content-derived, *not* a sequential index, so a puzzle
  keeps the same identity even if the bundle is regenerated or reordered. This
  is the contract Phase 3 relies on: date → puzzle mapping and the archive's
  per-puzzle cleared-status persistence key off `id`, so the pool can grow
  without scrambling players' history.
- **`cols`, `rows`, `inside`, `hideClues`** — the existing `Puzzle` fields.
  `presetActive` is tutorial-only and is omitted/empty for generated puzzles.
  **Phase 3 decode note:** the bundle emits `inside`/`hideClues` as `[[Int]]`
  (`[[c, r], …]`), whereas the app's `Puzzle` is `Codable`-synthesized over
  `Set<Cell>` and would expect `[{"c":…,"r":…}]`. So Phase 3 will need a small
  custom decoder (or an init from `[[Int]]`) — the formats are intentionally
  close but not byte-for-byte `Decodable`-compatible.
- **`meta`** — generation provenance, not used at runtime: `shownClueCount`,
  `rulesFired` (clue/dot tallies), and the `seed` that produced the puzzle.

```json
{
  "version": 1,
  "tiers": {
    "sprout":    [
      {
        "id": "a3f9",
        "cols": 4, "rows": 6,
        "inside": [[1,0],[2,0]],
        "hideClues": [[0,1]],
        "meta": { "shownClueCount": 18, "rulesFired": { "clue": 30, "dot": 12 }, "seed": 1 }
      }
    ],
    "mycelium":  [],
    "ancient":   [],
    "oldGrowth": []
  }
}
```

## Testing

Swift Testing, in the package's test target (same convention as the solver).
Key invariant tests:

- **Region properties** — generated regions are simply-connected (outside
  flood-fill is one component → no holes), non-trivial (not empty, not the
  whole grid), and deterministic (same seed → same region).
- **Hole detection** — a hand-built region with a hole is rejected; a solid one
  is accepted.
- **Clue-hiding correctness** — after hiding, the puzzle is still `.unique`
  with `guesses == 0`, *and* the hiding is maximal (no remaining shown clue can
  be removed while staying pure-logic).
- **Core invariant** — every emitted puzzle, re-solved from only its
  `hideClues` mask, returns `.unique` with `guesses == 0`. Reuses the exact
  validation path from `mycogrid-validate`.
- **Determinism & identity** — same seed + count → byte-identical bundle; same
  region → same `id`, different region → different `id`; no two puzzles in the
  bundle share a region.

### Self-audit

The generator's output can be piped through the already-shipped
`mycogrid-validate` CLI, so the bundle is independently audited by the tool
already trusted for the pool audit. If `mycogrid-validate` ever disagrees with
the generator's own gate, that's a real bug to surface — not suppress (same
principle as the Mycelium #1 finding: real defects show as failures).

## Performance note

The greedy clue-hiding loop runs `solve` once per hide attempt, and the under-
clued performance limit documented for the solver (commit `5022c32`) applies:
puzzles that are very sparsely clued are slow to validate. Because generation
is an offline batch job run by hand, wall-clock time is not a hard constraint,
but the driver should log progress per tier so a slow run is visible rather
than appearing hung.

## Open questions for the plan

- Exact fill-fraction band and whether it should vary per tier.
- Hash function and truncation length for `id` (collision-safety vs brevity).
- Whether dedup is by exact region identity only, or also rejects reflections/
  rotations of an existing region (leaning: exact identity only for v1 —
  simplest, and reflections still play as distinct puzzles).
