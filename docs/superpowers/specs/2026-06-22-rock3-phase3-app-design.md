# Mycogrid Phase 3 — App-Side Daily + Archive — Design Spec

**Date:** 2026-06-22
**Big Rock:** #3 (Solver + generator + daily/archive) — Phase 3 (App-side) + folds in Phase 4 (persistence)
**Status:** Design approved, ready for implementation plan

## Context

Big Rock #3 Phases 1 (uniqueness solver) and 2 (generator) are complete and
merged to `mycogrid` main (`b95c357`). The `mycogrid-generate` CLI now emits a
validated, reproducible JSON bundle of pure-logic-solvable puzzles per tier,
each with a stable content-derived `id`. See
`2026-06-21-rock3-generator-design.md`.

This spec covers **Phase 3 — wiring the app to that bundle** and building the
daily + archive experience, replacing the hand-curated `PuzzleData` per-tier
lists and the "Sprout #1/#2/#3" cycling. It also folds in roadmap **Phase 4**
(persistence migration), because the archive's cleared-status storage *is* the
persistence rework, and there is no install base to migrate (Mycogrid is
pre-TestFlight), so doing the clean thing now costs nothing.

Naming note: the product is **Mycogrid**; the Xcode project/target and app
source folder are still named `rootline` internally — a known residual, not
renamed. Paths below say `rootline/` accordingly.

## Goals

- Load the generated JSON bundle as a bundled app resource.
- Map each real calendar date to exactly one puzzle, deterministically, in a
  way that survives the pool growing later.
- Surface "Today's grove" on Home and a browseable Archive that doubles as the
  completion calendar.
- Replace the difficulty picker and the endless next-puzzle cycling with the
  daily + archive model.
- Establish a single id-keyed source of truth for completion/stats.

## Non-goals (explicitly out of scope)

- **New grid sizes / a 5th tier.** The four existing tiers map across the week
  with gentle repeats. No "Sunday-huge" board.
- **Technique/difficulty grading within a size.** Deferred from Phase 2; not
  revived here. Difficulty is grid size, and grid size is day-of-week.
- **Sharing, accounts, cloud sync.** Off the roadmap.
- **Regenerating or extending the bundle.** The generator already exists; this
  spec consumes its output. A one-time generation run produces the committed
  bundle.
- **Migrating existing player stats.** There is no install base; we start
  fresh.

## Key decisions (settled in brainstorming)

1. **Day-of-week → size.** One puzzle per real calendar date. The weekday picks
   the tier (= grid size = the only difficulty axis the generator produces):

   | Day        | Tier       | Size  |
   |------------|------------|-------|
   | Mon, Tue   | Sprout     | 4×6   |
   | Wed, Thu   | Mycelium   | 5×7   |
   | Fri        | Ancient    | 6×9   |
   | Sat, Sun   | Old Growth | 7×10  |

   This is the NYT-crossword "harder later in the week" rhythm, but it falls
   out of the existing tier model for free. Sunday is *not* enlarged — it is
   Old Growth, framed (if at all) as a roomy, unhurried one, in keeping with the
   calm north star.

2. **Daily rollover at local midnight.** "Today" follows the device's local
   calendar day. No server, no timezone coordination (no sharing to coordinate).

3. **No difficulty picker.** The archive *is* the difficulty selector: because
   each weekday is a different size and the archive reaches into the past, any
   size is always reachable by picking an appropriate past day. `DifficultyView`
   and the Home Difficulty card are removed.

4. **Deterministic, append-only date → puzzle mapping** (see Architecture).
   Stable as the pool grows. No "hash mod count."

5. **Cleared status keyed by stable content `id`,** never by date or array
   index — so history survives pool growth and bundle regeneration/reordering.

6. **Bundle runway, frequency-balanced ~2 years:** Sprout 200 · Mycelium 200 ·
   Ancient 100 · Old Growth 200 (~700 puzzles). Ancient occurs once a week so
   100 matches the others' ~2-year horizon. Appendable later without disturbing
   existing dates.

7. **Fresh-start persistence, single id-keyed source of truth.** A new
   `CompletionStore` keyed by `id` records every clear; Stats, Archive, and
   streak all derive from it. The old per-tier `ScoreStore` bookkeeping is
   retired.

8. **Backfillable/forgiving streak.** Consecutive cleared calendar days,
   repairable by solving a missed day from the archive. No reset, no "broke your
   streak" moment.

## Architecture

### Bundle loading + decode bridge

The committed bundle (`rootline/Resources/puzzles.json`) is added to the
`rootline` app target as a bundled resource. A `PuzzleBundle` Codable type
mirrors the JSON exactly — including `inside`/`hideClues` as `[[Int]]`
(`[[c, r], …]`) and the per-tier grouping — and exposes a mapping into the
app's existing `Puzzle` value type.

This bridge is the documented decode gotcha: the app's `Puzzle` is Codable over
`Set<Cell>` and expects `[{"c":…,"r":…}]`, whereas the bundle emits `[[Int]]`.
`Puzzle` already has an `init(cols:rows:inside:hide:presetActive:)` that takes
`[[Int]]`, so the bridge is a thin per-entry map, not a custom `Decodable`
conformance on `Puzzle` itself.

A loaded bundle entry retains its `id` (needed as the completion key). Define a
small `DailyPuzzle` (or `BundledPuzzle`) struct pairing `id: String` with the
mapped `Puzzle` and its `Tier`, so the rest of the app carries the id alongside
the playable puzzle.

The bundle is loaded once at launch and held by a `DailyService` (see below).
Tutorial `lessons` in `PuzzleData` are unaffected — they remain hand-authored.
Only the four per-tier *shipping* lists (`sprout`/`mycelium`/`ancient`/
`oldGrowth`) are removed from `PuzzleData`; their consumers move to
`DailyService`.

### DailyService — date → puzzle

`DailyService` owns the loaded bundle and the deterministic mapping. Pure
functions, easily unit-tested with injected dates.

- `mappingEpoch`: a fixed code constant — a chosen Monday (e.g. 2026-01-05) —
  anchoring the weekday math. Its value only needs to be ≤ every player's
  archive floor; it is otherwise arbitrary.
- `tier(for date) -> Tier`: from the weekday, per the table above.
- `puzzle(for date) -> DailyPuzzle`:
  1. `t = tier(for: date)`
  2. `n = number of days in [mappingEpoch, date] whose weekday maps to t`
     (the 0-based occurrence index of this tier-day since the epoch)
  3. `pool = bundle.puzzles(for: t)` (array, in committed/append order)
  4. return `pool[n % pool.count]`

**Append-only invariant:** the mapping depends on array *position*. The bundle
is only ever appended to; existing entries are never reordered or removed. As
long as the pool is topped up before `n` reaches `pool.count` (the ~2-year
runway), no already-played date is ever remapped. Cleared status keyed by `id`
provides defense-in-depth: even if a regeneration reordered entries, a player's
completions follow the puzzle content, not its slot.

### Home + play flow

- **Home** loses the Difficulty card. It gains a **"Today's grove"** card:
  weekday · formatted date · tier label · a **Play today** CTA. Once today's
  puzzle is cleared, the card shows a quiet cleared state (e.g. "Cleared · 1:23"
  whisper, consistent with the calm WinCard pattern), and the CTA can offer
  Archive / replay.
- **PlayView** is entered with a `DailyPuzzle` (id + puzzle + tier) and the
  calendar date it represents. On clear, it records the completion by `id`.
- The endless `nextPuzzle()` cycling is removed. After a win there is no "next"
  treadmill; the player returns to Home or the Archive. (WinCard's "next"
  affordance is replaced by "Back to archive" / "Done".)

### Archive — the completion calendar

A scrollable calendar (month sections, or continuous weeks — a UI detail for the
plan), from the archive floor up to today.

- **Archive floor:** set once, at first app open, to `firstOpenDate − 14 days`,
  persisted, and never moved forward. Every player therefore starts with ~2
  weeks of variety (all four sizes present) and accumulates history forward; no
  one loses access to a day they played, and no one faces a months-deep
  backlog. (The deterministic mapping means past dates already have puzzles —
  the floor is purely where the UI stops scrolling.)
- **Each cell:** the date, a cleared/not indicator, and a subtle size tint so
  the weekly size rhythm is legible. "Today" is highlighted. Tapping a cell
  opens that date's puzzle via `DailyService` and plays it; clearing it records
  the completion by `id` (and can repair a streak gap).
- **Streak:** current run of consecutive cleared calendar days ending at today
  (or, while today is still unsolved, ending at yesterday). Computed from the
  set of cleared dates derived from `CompletionStore`. Forgiving: solving a
  missed past day fills the gap and the streak recomputes upward. Surfaced
  quietly (a small status line), not as pressure.

This is the "completion calendar" the roadmap describes — meaningful only now
that puzzles have stable date-based identities.

### Persistence — fresh start, one source of truth

- **`CompletionStore`** (new). Key: puzzle `id`. Value: `{ tier, clearedDate,
  bestSeconds }`. Persisted in UserDefaults (same mechanism as today's stores).
  This is the single source of truth. From it we derive:
  - **Archive:** for any date, map date → `DailyPuzzle` → look up its `id` →
    cleared?
  - **Streak:** the set of cleared *dates* (a clear's date is recoverable by
    re-deriving which date(s) map to that id within the archive window, or by
    storing the played date alongside — see plan; storing the played date in the
    record is simplest and avoids ambiguity when an id recurs after a wrap).
  - **Stats screen:** per-tier cleared counts and best times, and the grand
    total — all reductions over `CompletionStore`.
- **`ScoreStore` retired.** Its `rootline_stats_v1` per-tier bookkeeping is
  removed; `StatsView` reads derived values from `CompletionStore` instead.
- **`ProgressStore` re-keyed.** The in-progress snapshot moves from
  `tier + groveNumber` identity to the puzzle `id` (plus the date being played,
  so resume returns to the right archive context). `Board(restoring:)` and
  `snapshot()` update accordingly.

**Note on streak vs. id recurrence:** because the pool can wrap after ~2 years,
the same `id` could in principle back two different dates far apart. Storing the
**played date** explicitly in each completion record (rather than reconstructing
it from the id) keeps the archive and streak unambiguous. `bestSeconds` still
tracks the fastest clear of that puzzle content. This is the recommended shape;
the plan finalizes the record fields.

## Components (units, each independently testable)

- **`PuzzleBundle`** — Codable JSON mirror + `→ [Tier: [DailyPuzzle]]` mapping.
  Depends on: `Puzzle`, `Tier`, `Cell`. Pure.
- **`DailyService`** — owns the loaded bundle; `tier(for:)`, `puzzle(for:)`,
  archive-window enumeration. Depends on: `PuzzleBundle`, a clock/calendar.
  Pure given an injected date.
- **`CompletionStore`** — id-keyed completion records + derived queries
  (cleared? per id, streak from dates, per-tier/total stats). Depends on:
  UserDefaults. Side-effecting only at the persistence boundary.
- **`ArchiveView`** — calendar UI over `DailyService` + `CompletionStore`.
- **"Today's grove" Home card** — small view over `DailyService` +
  `CompletionStore`.
- **`AppState`** — wires the above: `startToday()`, `startArchived(date:)`,
  `recordClear(...)`, replacing `startGame(tier:)` / `nextPuzzle()`.

## Data flow

```
launch
  └─ load puzzles.json → PuzzleBundle → DailyService (held by AppState)

Home "Today's grove"
  └─ DailyService.puzzle(for: today) → DailyPuzzle
       └─ CompletionStore.isCleared(id) → card state

Play (today or archived date)
  └─ Board(DailyPuzzle.puzzle, tier) ... on solve →
       CompletionStore.record(id, tier, date, seconds)
         └─ updates Archive cells, streak, Stats

Archive
  └─ for each date in [floor … today]:
       DailyService.puzzle(for: date) → id → CompletionStore.isCleared(id)
  └─ streak = longest tail run of consecutive cleared dates ending today/yesterday
```

## Error handling

- **Missing/corrupt bundle:** a bundled resource that fails to load or decode is
  a build/packaging defect, not a runtime-recoverable state. Fail loudly in
  DEBUG (assert) and degrade to a clear, non-crashing "couldn't load puzzles"
  surface in release rather than a blank board.
- **Empty pool for a tier:** treated as the same packaging defect (the generator
  guarantees non-empty tiers); guard the `% pool.count` against divide-by-zero
  with an explicit precondition surfaced in tests.
- **Date before the archive floor / before epoch:** the UI never offers these;
  `DailyService` mapping is still total (math works for any date ≥ epoch), but
  the archive enumeration clamps to the floor.

## Testing

Swift Testing (matching the solver/app conventions).

- **Mapping determinism:** `puzzle(for:)` returns the same id for the same date
  across runs; weekday → tier table is exhaustively correct.
- **Append-only stability:** appending entries to a tier pool does not change
  the puzzle returned for any date whose `n < oldCount`.
- **Occurrence-index math:** `n` counts tier-days correctly across week/month/
  year boundaries and DST transitions (local-calendar based).
- **Decode bridge:** a known bundle JSON decodes to the expected `Puzzle`
  values (inside/hideClues sets match); round-trips the id.
- **CompletionStore:** record/isCleared/best-time/per-tier counts/total;
  streak computation including the backfill-repairs-a-gap case and the
  today-still-unsolved (streak ends yesterday) case.
- **Archive enumeration:** floor is `firstOpen − 14d`, set once, never moves
  forward; window clamps correctly.
- **Bundle integrity (offline):** the committed `puzzles.json` re-validates
  through `mycogrid-validate` (every entry `.unique`, `guesses == 0`) — guards
  against a hand-edited or stale bundle, same drift-guard spirit as the tokens
  `check:tokens` step.

## Open questions for the plan

- Archive UI shape: month-sectioned grid vs. continuous week rows; how the size
  tint reads against the calm palette.
- Exact `CompletionStore` record fields and UserDefaults key/versioning
  (`rootline_completions_v1`?), and how `StatsView` reduces over it.
- Where the bundle generation run is recorded/reproducible (seed + counts
  committed alongside `puzzles.json` for the drift-guard).
- WinCard copy/affordances once "next" is gone (today vs. archived replay).
- Whether "Today's grove" cleared-state offers a replay or just a quiet marker.
