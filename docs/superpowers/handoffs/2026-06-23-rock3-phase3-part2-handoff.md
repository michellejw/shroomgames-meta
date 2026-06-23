# Mycogrid Rock #3 Phase 3 — Part 2 Handoff (Xcode/SwiftUI cutover)

**Date:** 2026-06-23
**For:** the session (likely the Xcode agent) executing Part 2 — Tasks 6–11.
**Status:** Part 1 (headless core) complete, reviewed, NOT merged. Part 2 not started.

## What this is

Big Rock #3 Phase 3 wires the Mycogrid app to the generated puzzle bundle as
NYT-style daily puzzles + a browseable archive, replacing the difficulty picker
and hand-curated cycling. The work is split:

- **Part 1 — headless core** (DONE): decode bridge, deterministic date→puzzle
  mapping, archive/streak, id-keyed completion store. TDD'd via `swift test` in
  the `MycogridSolver` SwiftPM package.
- **Part 2 — app/UI cutover** (THIS HANDOFF): generate the bundle, wire it into
  the app, build the Home "Today's grove" + `ArchiveView`, cut play/win/stats
  over to the new model, then remove the old model. GUI-bound: each task ends in
  an Xcode build + simulator check.

## Source documents (read these)

- **Spec:** `shroomgames-meta/docs/superpowers/specs/2026-06-22-rock3-phase3-app-design.md`
- **Plan (your task list — Tasks 6–11):** `shroomgames-meta/docs/superpowers/plans/2026-06-22-rock3-phase3-app.md`
- **Roadmap context:** `shroomgames-meta/ROADMAP.md` (Big Rock #3, Phase 3)

The plan's **Global Constraints** section is binding for every task. Tasks 6–11
each contain exact file paths and complete code.

## Repo / branch state

- Repo: `~/dev/games/shroom-games/mycogrid` (the app target/folder is still named
  `rootline` internally — a deliberate residual; do NOT rename. Product name is
  **Mycogrid**; never "Slitherlink").
- Branch: **`feature/rock3-phase3-app`** — check it out; do not work on `main`.
- Part 1 commits: `b95c357..c27070b` (5 tasks + 1 review fix). All headless tests
  green (XCTest 39 + Swift Testing 29).
- The branch is NOT merged. After Part 2 + final review, use
  superpowers:finishing-a-development-branch to merge.

## What Part 1 already gives you (interfaces Part 2 consumes)

All in `rootline/Daily/`, also symlinked into the package. All `public` where the
CLI needed them; the app target sees them directly.

- `struct DailyPuzzle { let id: String; let tier: Tier; let puzzle: Puzzle }`
- `struct PuzzleBundle` — `init(data: Data) throws`, `puzzles(for: Tier) -> [DailyPuzzle]`, `version`.
- `struct DailyService` — `init(bundle:calendar:)`; `tier(for: Date) -> Tier`;
  `puzzle(for: Date) -> DailyPuzzle?` (nil before the 2026-01-05 epoch or on an
  empty pool); `archiveDates(floor:today:) -> [Date]` (inclusive, most-recent-first);
  `currentStreak(today:isCleared:) -> Int` (backfillable); `static archiveFloor(firstOpen:calendar:) -> Date`.
- `@MainActor @Observable final class CompletionStore` — `init(defaults:)`,
  `isCleared(_:)`, `@discardableResult record(id:tier:seconds:) -> Bool` (true only
  on beating a pre-existing per-tier best), `totalCleared`, `clearedCount(for:)`,
  `bestSeconds(for:)`, `hasAnyStats`, `clearAll()`. UserDefaults key `rootline_completions_v1`.
- `mycogrid-validate bundle <path>` — audits a generated bundle (every entry
  `.unique`, `guesses==0`). Used in Task 6 to validate `puzzles.json`.

**Note:** `DailyService.live` (the `Bundle.main` loader) does NOT exist yet — you
create it in Task 7. `DailyService` is already shaped to accept it (injected
`bundle` + `calendar`).

## Part 2 task sequence (from the plan)

Each ends in a green Xcode build + a simulator verification (no headless test
target exists for the app). The app must compile after EVERY task — old types
(PuzzleData lists, ScoreStore, DifficultyView, startGame/nextPuzzle) are removed
only in the final cleanup (Task 11), so intermediate states keep building.

6. Generate `puzzles.json` (4 CLI runs + `jq` merge), validate via
   `mycogrid-validate bundle`, commit, add to the `rootline` target's Copy Bundle
   Resources (Xcode GUI step). Counts: **sprout 200 · mycelium 200 · ancient 100 ·
   oldGrowth 200**.
7. `AppState` daily wiring + `DailyService.live` loader + Home "Today's grove"
   card (replaces Difficulty card). Additive — old flow still compiles.
8. `ArchiveView` (completion calendar) + `.archive` screen + archive-floor
   persistence in `Settings`.
9. Play/win cutover: record by `id`, drop "Next puzzle" from WinCard/RevealedCard.
10. `StatsView` derives from `CompletionStore`.
11. Cleanup (atomic, only build-breaking-then-fixing task): remove PuzzleData
    per-tier lists + `puzzles(for:)` (keep `lessons`), delete `DifficultyView` +
    `ScoreStore`, remove `Settings.tier`, re-key `ProgressStore` to id + update
    `Board(restoring:using:)`, retire `auditPool()`/`pool` subcommand + `PoolTests`.

## Environment gotchas (learned during Part 1)

- Run `swift` commands from the repo root. Package path: `scripts/MycogridSolver`.
- **Stale `.build` cache:** if you hit spurious SwiftShims/module errors,
  `rm -rf scripts/MycogridSolver/.build` and retry (a known artifact in this repo).
- **SourceKit single-file noise:** new files in `rootline/Daily/` show "Cannot
  find type Tier/Puzzle" diagnostics when analyzed standalone (they reference
  sibling app files). This is expected until a file is in the Xcode target's
  compile set; the package `swift test` is the source of truth. Adding
  `puzzles.json` and the Daily files to the Xcode target (Tasks 6–7) resolves
  the app-target view.
- `Package.swift` already has `platforms: [.macOS(.v14)]` (needed for `@Observable`
  headless). Leave it.
- `Puzzle`/`Tier`/`PuzzleModel` are now `public` (Part 1 cascade). Additive — no
  app changes needed; just don't be surprised.

## Decisions/notes carried from the final review (NOT blockers)

- **Append-only is an operator rule.** When `puzzles.json` is regenerated/extended,
  only APPEND to a tier's array — never reorder/remove existing entries, or past
  dates remap. Task 6 writes `puzzles.README.md` stating this; keep it accurate.
- **Local-midnight rollover** via `.autoupdatingCurrent` is intended (offline,
  no accounts, no sharing). `DailyService.live` uses it.
- **Pool wrap** (~2 yr out) replays old puzzles, which read as pre-cleared and
  extend streaks. Acceptable; pools are sized to outrun the cursor. Top up
  (append) before then.
- Minor ergonomics: `DailyService.archiveFloor` is `static` (called at first-open
  before a service exists). Fine as-is.

## Open UI questions (decide during Part 2 visual review)

- Archive layout: month-sectioned vs continuous weeks; the per-tier tint reading
  calm against the palette (the plan uses `palette.pill`/`tierSelBg` tints —
  verify they read well).
- WinCard copy now that "Next puzzle" is gone (plan uses a single "Done").
- Confirm ShroomKit `ResultCard` supports a nil secondary button (Task 9 flags
  this to verify before editing; fall back to a primary-only initializer if not).

## Definition of done for Part 2

All 6 tasks complete and individually verified in the simulator; full app
regression (Home → Today → play → win/Done → Archive browse+play+streak → Stats →
resume-after-background); `mycogrid-validate bundle rootline/Resources/puzzles.json`
passes; then a final whole-branch review and merge via
superpowers:finishing-a-development-branch. Update `ROADMAP.md` Phase 3 + Phase 4
checkboxes on merge.
