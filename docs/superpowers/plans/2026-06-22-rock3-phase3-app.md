# Mycogrid Phase 3 — App-Side Daily + Archive — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire the Mycogrid app to the generated puzzle bundle as NYT-style daily puzzles plus a browseable archive, replacing the difficulty picker and the hand-curated per-tier cycling.

**Architecture:** A headless core (decode bridge, deterministic date→puzzle mapping, id-keyed completion store) is authored as app files under `rootline/Daily/`, symlinked into the existing `MycogridSolver` SwiftPM package so it is unit-tested headlessly via `swift test` (the established pattern for `Engine.swift`/`Tier.swift`). The SwiftUI/Xcode layer (Home "Today's grove", `ArchiveView`, `AppState` rewire) consumes that core and is verified by build + simulator.

**Tech Stack:** Swift 5.9, SwiftUI, Swift Testing (`import Testing`), the `MycogridSolver` SwiftPM package, `jq` for bundle merging, Xcode for the app target.

## Global Constraints

- Product name in all user-facing copy and docs is **Mycogrid**; the Xcode project/target and source folder are still named **`rootline`** internally — do not rename. Never "Slitherlink."
- Difficulty = grid size = day-of-week. No new grid sizes, no 5th `Tier`, no technique-grading.
- Day→tier table (calendar weekday): **Mon, Tue → sprout · Wed, Thu → mycelium · Fri → ancient · Sat, Sun → oldGrowth**.
- Date→puzzle mapping is **deterministic and append-only**: index by occurrence-count since a fixed epoch into the tier's pool; `% pool.count` on overflow. Pool arrays are never reordered. **No "hash mod count."**
- Cleared status is keyed by the puzzle's stable content **`id`** (12-char FNV-1a hex from `puzzleID(cols:rows:inside:)`), never by date or array index.
- Bundle runway: **sprout 200 · mycelium 200 · ancient 100 · oldGrowth 200**.
- Archive floor = `firstOpenDate − 14 days`, set once, never moved forward.
- Streak is **backfillable/forgiving** (consecutive cleared calendar days ending today/yesterday; repairable from the archive).
- Fresh-start persistence: one id-keyed `CompletionStore` is the source of truth; `ScoreStore` retired; no migration code.
- Calm/meditative north star: no arcade framing, streak surfaced quietly.
- New `UserDefaults` keys follow the `rootline_<name>_v<n>` convention.
- Existing app source of truth lives in `rootline/`; package consumes app files via symlink. Author new headless-core files in `rootline/Daily/` and add a matching symlink under `scripts/MycogridSolver/Sources/MycogridSolver/AppModel/`.

---

# Part 1 — Headless core (TDD via `swift test`)

All Part 1 work builds and tests with:
`swift test --package-path scripts/MycogridSolver`
Run it from the repo root: `/Users/michelleweirathmueller/dev/games/shroom-games/mycogrid`.

Tests use `@testable import MycogridSolver` (the app types are `internal`).

---

## Task 1: Decode bridge — `DailyPuzzle` + `PuzzleBundle`

**Files:**
- Create: `rootline/Daily/PuzzleBundle.swift`
- Create symlink: `scripts/MycogridSolver/Sources/MycogridSolver/AppModel/PuzzleBundle.swift` → `../../../../../rootline/Daily/PuzzleBundle.swift`
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/PuzzleBundleTests.swift`

**Interfaces:**
- Consumes: `Puzzle` (`init(cols:rows:inside:hide:presetActive:)`), `Cell`, `Tier`.
- Produces:
  - `struct DailyPuzzle: Hashable, Sendable { let id: String; let tier: Tier; let puzzle: Puzzle }`
  - `struct PuzzleBundle: Sendable` with `init(byTier:version:)`, `init(data: Data) throws`, `func puzzles(for: Tier) -> [DailyPuzzle]`, `let version: Int`.
  - `enum PuzzleBundleError: Error, Equatable { case unknownTier(String) }`

- [ ] **Step 1: Write the failing test**

```swift
// scripts/MycogridSolver/Tests/MycogridSolverTests/PuzzleBundleTests.swift
import Testing
import Foundation
@testable import MycogridSolver

@Suite struct PuzzleBundleTests {
    // [c,r] pairs, matching the generator's emit order.
    static let json = """
    {
      "version": 1,
      "tiers": {
        "sprout": [
          { "id": "abc123",
            "cols": 4, "rows": 6,
            "inside": [[1,0],[2,0]],
            "hideClues": [[0,1]],
            "meta": { "shownClueCount": 18, "rulesFired": { "clue": 30, "dot": 12 }, "seed": 1 } }
        ],
        "mycelium": []
      }
    }
    """.data(using: .utf8)!

    @Test func decodesEntriesIntoDailyPuzzles() throws {
        let bundle = try PuzzleBundle(data: Self.json)
        #expect(bundle.version == 1)
        let sprout = bundle.puzzles(for: .sprout)
        #expect(sprout.count == 1)
        let p = sprout[0]
        #expect(p.id == "abc123")
        #expect(p.tier == .sprout)
        #expect(p.puzzle.cols == 4 && p.puzzle.rows == 6)
        #expect(p.puzzle.inside == Set([Cell(c: 1, r: 0), Cell(c: 2, r: 0)]))
        #expect(p.puzzle.hideClues == Set([Cell(c: 0, r: 1)]))
    }

    @Test func emptyTierAndAbsentTierReturnEmpty() throws {
        let bundle = try PuzzleBundle(data: Self.json)
        #expect(bundle.puzzles(for: .mycelium).isEmpty)
        #expect(bundle.puzzles(for: .oldGrowth).isEmpty)
    }

    @Test func unknownTierKeyThrows() {
        let bad = #"{ "version": 1, "tiers": { "bogus": [] } }"#.data(using: .utf8)!
        #expect(throws: PuzzleBundleError.unknownTier("bogus")) {
            try PuzzleBundle(data: bad)
        }
    }

    @Test func preservesArrayOrder() throws {
        let two = """
        { "version": 1, "tiers": { "sprout": [
          { "id": "first",  "cols": 4, "rows": 6, "inside": [[0,0]], "hideClues": [],
            "meta": { "shownClueCount": 1, "rulesFired": {"clue":1,"dot":0}, "seed": 1 } },
          { "id": "second", "cols": 4, "rows": 6, "inside": [[1,0]], "hideClues": [],
            "meta": { "shownClueCount": 1, "rulesFired": {"clue":1,"dot":0}, "seed": 1 } }
        ] } }
        """.data(using: .utf8)!
        let ids = try PuzzleBundle(data: two).puzzles(for: .sprout).map(\.id)
        #expect(ids == ["first", "second"])
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter PuzzleBundleTests`
Expected: FAIL — `cannot find 'PuzzleBundle' in scope`.

- [ ] **Step 3: Write the implementation**

```swift
// rootline/Daily/PuzzleBundle.swift
import Foundation

/// One playable puzzle from the generated bundle, carrying its stable id and tier.
struct DailyPuzzle: Hashable, Sendable {
    let id: String
    let tier: Tier
    let puzzle: Puzzle
}

/// Codable mirror of the generator's `puzzles.json`. `inside`/`hideClues` arrive
/// as `[[c, r]]` pairs (not `[{c,r}]`); the bridge below maps them into `Puzzle`.
/// The generator's `meta` block is provenance-only and intentionally ignored
/// here (unknown JSON keys are dropped by the decoder).
private struct PuzzleBundleJSON: Decodable {
    let version: Int
    let tiers: [String: [Entry]]
    struct Entry: Decodable {
        let id: String
        let cols: Int
        let rows: Int
        let inside: [[Int]]
        let hideClues: [[Int]]
    }
}

enum PuzzleBundleError: Error, Equatable { case unknownTier(String) }

/// The loaded bundle: per-tier ordered arrays of playable puzzles. Array order is
/// the committed, append-only order the date→puzzle mapping depends on.
struct PuzzleBundle: Sendable {
    let version: Int
    private let byTier: [Tier: [DailyPuzzle]]

    init(byTier: [Tier: [DailyPuzzle]], version: Int = 1) {
        self.byTier = byTier
        self.version = version
    }

    func puzzles(for tier: Tier) -> [DailyPuzzle] { byTier[tier] ?? [] }

    /// Decode from the generator's JSON. Throws `PuzzleBundleError.unknownTier`
    /// for an unrecognized tier key, or a `DecodingError` for malformed JSON.
    init(data: Data) throws {
        let json = try JSONDecoder().decode(PuzzleBundleJSON.self, from: data)
        var out: [Tier: [DailyPuzzle]] = [:]
        for (key, entries) in json.tiers {
            guard let tier = Tier(rawValue: key) else {
                throw PuzzleBundleError.unknownTier(key)
            }
            out[tier] = entries.map { e in
                DailyPuzzle(
                    id: e.id,
                    tier: tier,
                    puzzle: Puzzle(cols: e.cols, rows: e.rows, inside: e.inside, hide: e.hideClues)
                )
            }
        }
        self.init(byTier: out, version: json.version)
    }
}
```

Create the symlink:

```bash
cd /Users/michelleweirathmueller/dev/games/shroom-games/mycogrid
mkdir -p rootline/Daily
# (the .swift file above is created by the editor; then:)
ln -s ../../../../../rootline/Daily/PuzzleBundle.swift \
  scripts/MycogridSolver/Sources/MycogridSolver/AppModel/PuzzleBundle.swift
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter PuzzleBundleTests`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add rootline/Daily/PuzzleBundle.swift \
  scripts/MycogridSolver/Sources/MycogridSolver/AppModel/PuzzleBundle.swift \
  scripts/MycogridSolver/Tests/MycogridSolverTests/PuzzleBundleTests.swift
git commit -m "feat: add PuzzleBundle decode bridge for generated puzzles.json"
```

---

## Task 2: `mycogrid-validate bundle <path>` — bundle integrity audit

This gives the spec's drift/integrity guard: every committed bundle entry re-solves `.unique` with `guesses == 0`. Reuses `PuzzleBundle` + the existing solver path from `auditPuzzle`.

**Files:**
- Create: `scripts/MycogridSolver/Sources/MycogridSolver/BundleAudit.swift`
- Modify: `scripts/MycogridSolver/Sources/mycogrid-validate/main.swift` (add a `bundle` case)
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/BundleAuditTests.swift`

**Interfaces:**
- Consumes: `PuzzleBundle`, `Puzzle`, `PuzzleModel`, `solve(_:)`, `PuzzleClues`, `Verdict`.
- Produces: `func auditBundle(_ bundle: PuzzleBundle) -> [PoolReport]` (reuses the existing `PoolReport` type from `Pool.swift`).

- [ ] **Step 1: Write the failing test**

```swift
// scripts/MycogridSolver/Tests/MycogridSolverTests/BundleAuditTests.swift
import Testing
import Foundation
@testable import MycogridSolver

@Suite struct BundleAuditTests {
    /// Build a one-entry bundle from a known-good hand puzzle (Sprout grove #1,
    /// all clues shown) so the audit must pass.
    @Test func auditPassesForAValidEntry() {
        let p = Puzzle(cols: 4, rows: 6, inside: [
            [1,0],[2,0],
            [0,1],[1,1],[2,1],[3,1],
            [0,2],[1,2],[2,2],[3,2],
            [0,3],[1,3],[2,3],[3,3],
            [0,4],[1,4],[2,4],[3,4],
            [1,5],[2,5]
        ])
        let dp = DailyPuzzle(id: "x", tier: .sprout, puzzle: p)
        let bundle = PuzzleBundle(byTier: [.sprout: [dp]])
        let reports = auditBundle(bundle)
        #expect(reports.count == 1)
        #expect(reports[0].passed)
        #expect(reports[0].guesses == 0)
    }

    @Test func auditFlagsANonUniqueEntry() {
        // Two separate single cells far apart with all clues hidden → not a
        // single uniquely-determined loop. Expect non-pass.
        let p = Puzzle(cols: 4, rows: 6, inside: [[0,0]], hide: [[0,0]])
        let dp = DailyPuzzle(id: "y", tier: .sprout, puzzle: p)
        let reports = auditBundle(PuzzleBundle(byTier: [.sprout: [dp]]))
        #expect(reports[0].passed == false)
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter BundleAuditTests`
Expected: FAIL — `cannot find 'auditBundle' in scope`.

- [ ] **Step 3: Write the implementation**

```swift
// scripts/MycogridSolver/Sources/MycogridSolver/BundleAudit.swift
import Foundation

/// Re-validate every entry in a loaded bundle through the solver: each must be
/// `.unique` with `guesses == 0` and its solver loop must match the region's
/// derived loop. Mirrors `auditPuzzle` but labels by id.
public func auditBundle(_ bundle: PuzzleBundle) -> [PoolReport] {
    var out: [PoolReport] = []
    for tier in Tier.allCases {
        for dp in bundle.puzzles(for: tier) {
            out.append(auditPuzzle(dp.puzzle, label: "\(tier.rawValue):\(dp.id)"))
        }
    }
    return out
}
```

`auditBundle` is `public` and `auditPuzzle` is already `internal` in the same module — both visible from the CLI target via `import MycogridSolver` for the public one. Add the `bundle` case to the CLI:

```swift
// scripts/MycogridSolver/Sources/mycogrid-validate/main.swift
// Update the usage string on line 15:
//   fail("usage: mycogrid-validate <pool | bundle <path> | file <path>>", code: 2)
// Add this case to the switch (after the `pool` case):

case "bundle":
    guard args.count >= 3 else { fail("usage: mycogrid-validate bundle <path>", code: 2) }
    let bundle: PuzzleBundle
    do {
        let data = try Data(contentsOf: URL(fileURLWithPath: args[2]))
        bundle = try PuzzleBundle(data: data)
    } catch {
        fail("error reading \(args[2]): \(error)", code: 2)
    }
    let reports = auditBundle(bundle)
    print("\(pad("tier:id", 26)) \(pad("verdict", 9)) \(pad("guesses", 8)) \(pad("match", 6)) \(pad("oracle", 7)) result")
    var anyFail = false
    for r in reports {
        if !r.passed { anyFail = true }
        print("\(pad(r.label, 26)) \(pad(r.verdict.rawValue, 9)) \(pad(String(r.guesses), 8)) \(pad(r.matchesStored ? "yes" : "no", 6)) \(pad(r.oracleOK ? "yes" : "no", 7)) \(r.passed ? "PASS" : "FAIL")")
    }
    print("\(reports.count) entries, \(reports.filter { !$0.passed }.count) failing")
    exit(anyFail ? 1 : 0)
```

Note: `PuzzleBundle(data:)` is `internal` (Task 1) and the CLI does `import MycogridSolver`, so `PuzzleBundle` must be reachable from the CLI. The CLI already uses internal-ish helpers via the module's public surface; make `PuzzleBundle`, its `init(data:)`, `puzzles(for:)`, `version`, `DailyPuzzle`, and `PuzzleBundleError` **`public`** (and their stored inits) so the executable target can use them. Update Task 1's types to `public` where the CLI touches them: `public struct PuzzleBundle`, `public init(data:) throws`, `public func puzzles(for:)`, `public struct DailyPuzzle` + its memberwise needs `public init`. (App + tests are unaffected by widening to `public`.)

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter BundleAuditTests`
Expected: PASS (2 tests). Also confirm the CLI builds:
Run: `swift build --package-path scripts/MycogridSolver`
Expected: Build complete.

- [ ] **Step 5: Commit**

```bash
git add scripts/MycogridSolver/Sources/MycogridSolver/BundleAudit.swift \
  scripts/MycogridSolver/Sources/mycogrid-validate/main.swift \
  scripts/MycogridSolver/Sources/MycogridSolver/AppModel/PuzzleBundle.swift \
  rootline/Daily/PuzzleBundle.swift \
  scripts/MycogridSolver/Tests/MycogridSolverTests/BundleAuditTests.swift
git commit -m "feat: add 'mycogrid-validate bundle <path>' integrity audit"
```

---

## Task 3: `DailyService` — date → tier → puzzle

**Files:**
- Create: `rootline/Daily/DailyService.swift`
- Create symlink: `scripts/MycogridSolver/Sources/MycogridSolver/AppModel/DailyService.swift` → `../../../../../rootline/Daily/DailyService.swift`
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/DailyServiceTests.swift`

**Interfaces:**
- Consumes: `PuzzleBundle`, `DailyPuzzle`, `Tier`.
- Produces (this task):
  - `struct DailyService: Sendable { let bundle: PuzzleBundle; var calendar: Calendar; init(bundle:calendar:) }`
  - `static let mappingEpoch: DateComponents` (a Monday)
  - `func tier(for: Date) -> Tier`
  - `func tierOccurrenceIndex(for: Date, tier: Tier) -> Int`
  - `func puzzle(for: Date) -> DailyPuzzle?`
  - `static func archiveFloor(firstOpen: Date, calendar: Calendar) -> Date`

- [ ] **Step 1: Write the failing test**

```swift
// scripts/MycogridSolver/Tests/MycogridSolverTests/DailyServiceTests.swift
import Testing
import Foundation
@testable import MycogridSolver

@Suite struct DailyServiceTests {
    // A UTC calendar makes weekday math deterministic in tests.
    static var utc: Calendar {
        var c = Calendar(identifier: .gregorian)
        c.timeZone = TimeZone(identifier: "UTC")!
        return c
    }
    static func day(_ y: Int, _ m: Int, _ d: Int) -> Date {
        utc.date(from: DateComponents(year: y, month: m, day: d))!
    }

    /// Build a bundle with N distinctly-id'd placeholder puzzles per tier so we
    /// can read back which index a date selected. (Solver validity is not under
    /// test here — only the mapping.)
    static func bundle(perTier n: Int) -> PuzzleBundle {
        var byTier: [Tier: [DailyPuzzle]] = [:]
        for tier in Tier.allCases {
            byTier[tier] = (0..<n).map { i in
                DailyPuzzle(id: "\(tier.rawValue)-\(i)", tier: tier,
                            puzzle: Puzzle(cols: tier.cols, rows: tier.rows, inside: [[0,0]]))
            }
        }
        return PuzzleBundle(byTier: byTier)
    }

    func svc(_ n: Int = 300) -> DailyService {
        DailyService(bundle: Self.bundle(perTier: n), calendar: Self.utc)
    }

    @Test func weekdayMapsToTier() {
        let s = svc()
        // 2026-06-22 is a Monday.
        #expect(s.tier(for: Self.day(2026, 6, 22)) == .sprout)     // Mon
        #expect(s.tier(for: Self.day(2026, 6, 23)) == .sprout)     // Tue
        #expect(s.tier(for: Self.day(2026, 6, 24)) == .mycelium)   // Wed
        #expect(s.tier(for: Self.day(2026, 6, 25)) == .mycelium)   // Thu
        #expect(s.tier(for: Self.day(2026, 6, 26)) == .ancient)    // Fri
        #expect(s.tier(for: Self.day(2026, 6, 27)) == .oldGrowth)  // Sat
        #expect(s.tier(for: Self.day(2026, 6, 28)) == .oldGrowth)  // Sun
    }

    @Test func epochMondayIsSproutOccurrenceZero() {
        let s = svc()
        // mappingEpoch = 2026-01-05 (Mon) → first sprout day, index 0.
        let p = s.puzzle(for: Self.day(2026, 1, 5))
        #expect(p?.id == "sprout-0")
    }

    @Test func occurrenceIndexCountsOnlyMatchingWeekdays() {
        let s = svc()
        // Tue 2026-01-06 is the 2nd sprout day since epoch → index 1.
        #expect(s.tierOccurrenceIndex(for: Self.day(2026, 1, 6), tier: .sprout) == 1)
        // Next Mon 2026-01-12 is the 3rd sprout day → index 2.
        #expect(s.tierOccurrenceIndex(for: Self.day(2026, 1, 12), tier: .sprout) == 2)
        // First ancient day (Fri 2026-01-09) → index 0.
        #expect(s.tierOccurrenceIndex(for: Self.day(2026, 1, 9), tier: .ancient) == 0)
    }

    @Test func mappingIsDeterministic() {
        let s = svc()
        let a = s.puzzle(for: Self.day(2026, 6, 22))?.id
        let b = s.puzzle(for: Self.day(2026, 6, 22))?.id
        #expect(a == b)
    }

    @Test func poolCyclesWithModulo() {
        // Only 1 sprout puzzle → every sprout day maps to index 0.
        let s = DailyService(bundle: Self.bundle(perTier: 1), calendar: Self.utc)
        #expect(s.puzzle(for: Self.day(2026, 1, 5))?.id == "sprout-0")
        #expect(s.puzzle(for: Self.day(2026, 1, 6))?.id == "sprout-0")
    }

    @Test func appendOnlyStabilityHoldsBeforeWrap() {
        // A larger pool must not change the puzzle for a date whose index < oldCount.
        let small = DailyService(bundle: Self.bundle(perTier: 10), calendar: Self.utc)
        let big   = DailyService(bundle: Self.bundle(perTier: 50), calendar: Self.utc)
        let d = Self.day(2026, 1, 12)   // sprout index 2 (< 10)
        #expect(small.puzzle(for: d)?.id == big.puzzle(for: d)?.id)
    }

    @Test func emptyPoolReturnsNil() {
        let s = DailyService(bundle: PuzzleBundle(byTier: [:]), calendar: Self.utc)
        #expect(s.puzzle(for: Self.day(2026, 6, 22)) == nil)
    }

    @Test func archiveFloorIsFourteenDaysBefore() {
        let floor = DailyService.archiveFloor(firstOpen: Self.day(2026, 6, 22), calendar: Self.utc)
        #expect(floor == Self.day(2026, 6, 8))
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter DailyServiceTests`
Expected: FAIL — `cannot find 'DailyService' in scope`.

- [ ] **Step 3: Write the implementation**

```swift
// rootline/Daily/DailyService.swift
import Foundation

/// Maps real calendar dates to puzzles and (Task 4) enumerates the archive
/// window. Pure given an injected `Calendar`; no I/O. The live app loads the
/// bundle from `Bundle.main` via `DailyService.live` (Part 2).
struct DailyService: Sendable {
    let bundle: PuzzleBundle
    var calendar: Calendar

    /// Fixed math anchor — a Monday. Its only requirement is to be ≤ every
    /// player's archive floor; otherwise arbitrary.
    static let mappingEpoch = DateComponents(year: 2026, month: 1, day: 5) // Monday

    init(bundle: PuzzleBundle, calendar: Calendar = .autoupdatingCurrent) {
        self.bundle = bundle
        self.calendar = calendar
    }

    /// Tier (grid size) for a date, from its weekday.
    /// Calendar weekday: 1=Sun, 2=Mon … 7=Sat.
    func tier(for date: Date) -> Tier {
        switch calendar.component(.weekday, from: date) {
        case 2, 3: return .sprout      // Mon, Tue
        case 4, 5: return .mycelium    // Wed, Thu
        case 6:    return .ancient     // Fri
        default:   return .oldGrowth   // Sat (7), Sun (1)
        }
    }

    /// 0-based count of dates in [epoch, date) that map to `tier` — i.e. the
    /// index of `date` among that tier's days since the epoch.
    func tierOccurrenceIndex(for date: Date, tier: Tier) -> Int {
        let day = calendar.startOfDay(for: date)
        guard let epoch = calendar.date(from: Self.mappingEpoch) else { return 0 }
        let epochDay = calendar.startOfDay(for: epoch)
        let totalDays = calendar.dateComponents([.day], from: epochDay, to: day).day ?? 0
        guard totalDays > 0 else { return 0 }
        var count = 0
        for offset in 0..<totalDays {                 // exclude `date` itself → 0-based
            if let d = calendar.date(byAdding: .day, value: offset, to: epochDay),
               self.tier(for: d) == tier {
                count += 1
            }
        }
        return count
    }

    /// The puzzle for a date: the n-th occurrence of that tier's weekday since
    /// the epoch, indexed into the tier's append-only pool (cycling on overflow).
    func puzzle(for date: Date) -> DailyPuzzle? {
        let t = tier(for: date)
        let pool = bundle.puzzles(for: t)
        guard !pool.isEmpty else { return nil }
        let n = tierOccurrenceIndex(for: date, tier: t)
        return pool[n % pool.count]
    }

    /// Archive back-scroll floor: 14 days before first open, fixed once.
    static func archiveFloor(firstOpen: Date, calendar: Calendar = .autoupdatingCurrent) -> Date {
        calendar.date(byAdding: .day, value: -14, to: calendar.startOfDay(for: firstOpen))
            ?? calendar.startOfDay(for: firstOpen)
    }
}
```

```bash
ln -s ../../../../../rootline/Daily/DailyService.swift \
  scripts/MycogridSolver/Sources/MycogridSolver/AppModel/DailyService.swift
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter DailyServiceTests`
Expected: PASS (8 tests).

- [ ] **Step 5: Commit**

```bash
git add rootline/Daily/DailyService.swift \
  scripts/MycogridSolver/Sources/MycogridSolver/AppModel/DailyService.swift \
  scripts/MycogridSolver/Tests/MycogridSolverTests/DailyServiceTests.swift
git commit -m "feat: add DailyService date->tier->puzzle mapping"
```

---

## Task 4: `DailyService` — archive enumeration + backfillable streak

**Files:**
- Modify: `rootline/Daily/DailyService.swift` (append two methods)
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/DailyServiceTests.swift` (add a suite)

**Interfaces:**
- Consumes: Task 3's `DailyService`, `DailyPuzzle`.
- Produces:
  - `func archiveDates(floor: Date, today: Date) -> [Date]` — start-of-day dates, **most-recent-first**, inclusive.
  - `func currentStreak(today: Date, isCleared: (DailyPuzzle) -> Bool) -> Int`

- [ ] **Step 1: Write the failing test**

```swift
// Append to DailyServiceTests.swift
@Suite struct DailyServiceArchiveTests {
    typealias T = DailyServiceTests
    func svc(_ n: Int = 300) -> DailyService {
        DailyService(bundle: DailyServiceTests.bundle(perTier: n), calendar: DailyServiceTests.utc)
    }

    @Test func archiveDatesAreInclusiveAndDescending() {
        let s = svc()
        let dates = s.archiveDates(floor: T.day(2026, 6, 20), today: T.day(2026, 6, 22))
        #expect(dates == [T.day(2026, 6, 22), T.day(2026, 6, 21), T.day(2026, 6, 20)])
    }

    @Test func archiveDatesEmptyWhenTodayBeforeFloor() {
        let s = svc()
        #expect(s.archiveDates(floor: T.day(2026, 6, 22), today: T.day(2026, 6, 20)).isEmpty)
    }

    @Test func streakCountsConsecutiveClearedEndingToday() {
        let s = svc()
        // Cleared: 20, 21, 22 (today). Streak = 3.
        let cleared: Set<String> = [
            s.puzzle(for: T.day(2026, 6, 20))!.id,
            s.puzzle(for: T.day(2026, 6, 21))!.id,
            s.puzzle(for: T.day(2026, 6, 22))!.id,
        ]
        let n = s.currentStreak(today: T.day(2026, 6, 22)) { cleared.contains($0.id) }
        #expect(n == 3)
    }

    @Test func streakEndsYesterdayWhenTodayUnsolved() {
        let s = svc()
        // Cleared: 20, 21 but NOT 22. Today 22 unsolved → streak ends yesterday = 2.
        let cleared: Set<String> = [
            s.puzzle(for: T.day(2026, 6, 20))!.id,
            s.puzzle(for: T.day(2026, 6, 21))!.id,
        ]
        let n = s.currentStreak(today: T.day(2026, 6, 22)) { cleared.contains($0.id) }
        #expect(n == 2)
    }

    @Test func streakStopsAtAGap() {
        let s = svc()
        // Cleared: 22 (today) and 20, but 21 missing → streak = 1 (just today).
        let cleared: Set<String> = [
            s.puzzle(for: T.day(2026, 6, 22))!.id,
            s.puzzle(for: T.day(2026, 6, 20))!.id,
        ]
        let n = s.currentStreak(today: T.day(2026, 6, 22)) { cleared.contains($0.id) }
        #expect(n == 1)
    }

    @Test func streakIsZeroWhenNothingCleared() {
        let s = svc()
        #expect(s.currentStreak(today: T.day(2026, 6, 22)) { _ in false } == 0)
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter DailyServiceArchiveTests`
Expected: FAIL — `value of type 'DailyService' has no member 'archiveDates'`.

- [ ] **Step 3: Write the implementation**

```swift
// Append inside `struct DailyService` in rootline/Daily/DailyService.swift

/// Start-of-day dates from `floor` through `today` inclusive, most-recent-first.
func archiveDates(floor: Date, today: Date) -> [Date] {
    let f = calendar.startOfDay(for: floor)
    let t = calendar.startOfDay(for: today)
    guard t >= f else { return [] }
    let n = calendar.dateComponents([.day], from: f, to: t).day ?? 0
    let ascending = (0...n).compactMap { calendar.date(byAdding: .day, value: $0, to: f) }
    return ascending.reversed()
}

/// Current backfillable streak: consecutive cleared days ending at today, or at
/// yesterday when today is not cleared yet. Walks backward, stopping at the
/// first gap. `isCleared` is supplied by the completion store.
func currentStreak(today: Date, isCleared: (DailyPuzzle) -> Bool) -> Int {
    var day = calendar.startOfDay(for: today)
    // If today's puzzle isn't cleared, an in-progress today shouldn't break the
    // streak — start the count from yesterday.
    if let p = puzzle(for: day), !isCleared(p) {
        guard let yesterday = calendar.date(byAdding: .day, value: -1, to: day) else { return 0 }
        day = yesterday
    }
    var streak = 0
    while let p = puzzle(for: day), isCleared(p) {
        streak += 1
        guard let prev = calendar.date(byAdding: .day, value: -1, to: day) else { break }
        day = prev
    }
    return streak
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter DailyServiceArchiveTests`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add rootline/Daily/DailyService.swift \
  scripts/MycogridSolver/Tests/MycogridSolverTests/DailyServiceTests.swift
git commit -m "feat: add archive enumeration + backfillable streak to DailyService"
```

---

## Task 5: `CompletionStore` — id-keyed source of truth

**Files:**
- Create: `rootline/Daily/CompletionStore.swift`
- Create symlink: `scripts/MycogridSolver/Sources/MycogridSolver/AppModel/CompletionStore.swift` → `../../../../../rootline/Daily/CompletionStore.swift`
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/CompletionStoreTests.swift`

**Design note (refines the spec):** The spec flagged storing the *played date* to disambiguate id-recurrence after a pool wrap. It is unnecessary: cleared-ness is always evaluated **date → id → membership** (never id → date), so the archive and streak read correctly even when one id backs two far-apart dates (both show cleared). The record therefore stays minimal: `{ tier, bestSeconds }` keyed by id. "Groves cleared" counts distinct ids.

**Interfaces:**
- Consumes: `Tier`, `DailyPuzzle`.
- Produces:
  - `struct Completion: Codable, Equatable, Sendable { let tier: Tier; var bestSeconds: Int }`
  - `@MainActor @Observable final class CompletionStore` with: `init(defaults:)`, `private(set) var byID: [String: Completion]`, `func isCleared(_ id: String) -> Bool`, `func isCleared(_ p: DailyPuzzle) -> Bool`, `@discardableResult func record(id:tier:seconds:) -> Bool`, `var totalCleared: Int`, `func clearedCount(for: Tier) -> Int`, `func bestSeconds(for: Tier) -> Int?`, `var hasAnyStats: Bool`, `func clearAll()`.
- `record` returns `true` only when this clear beat a **pre-existing** per-tier best (the win-card whisper; first clear → `false`, matching today's `ScoreStore.record == .newBest`).

- [ ] **Step 1: Write the failing test**

```swift
// scripts/MycogridSolver/Tests/MycogridSolverTests/CompletionStoreTests.swift
import Testing
import Foundation
@testable import MycogridSolver

@MainActor @Suite struct CompletionStoreTests {
    func freshStore() -> CompletionStore {
        let suite = "test-completions"
        let d = UserDefaults(suiteName: suite)!
        d.removePersistentDomain(forName: suite)
        return CompletionStore(defaults: d)
    }

    @Test func recordsAndReportsCleared() {
        let s = freshStore()
        #expect(s.isCleared("a") == false)
        _ = s.record(id: "a", tier: .sprout, seconds: 90)
        #expect(s.isCleared("a"))
        #expect(s.totalCleared == 1)
        #expect(s.clearedCount(for: .sprout) == 1)
        #expect(s.bestSeconds(for: .sprout) == 90)
    }

    @Test func firstClearIsNotAWhisper() {
        let s = freshStore()
        #expect(s.record(id: "a", tier: .sprout, seconds: 90) == false)
    }

    @Test func fasterReclearOfTierWhispers() {
        let s = freshStore()
        _ = s.record(id: "a", tier: .sprout, seconds: 90)
        #expect(s.record(id: "b", tier: .sprout, seconds: 80) == true)   // beat 90
        #expect(s.bestSeconds(for: .sprout) == 80)
    }

    @Test func slowerReclearDoesNotWhisperOrRegressBest() {
        let s = freshStore()
        _ = s.record(id: "a", tier: .sprout, seconds: 80)
        #expect(s.record(id: "a", tier: .sprout, seconds: 95) == false)
        #expect(s.bestSeconds(for: .sprout) == 80)   // keeps fastest
        #expect(s.totalCleared == 1)                 // same id, still one
    }

    @Test func perTierIsolationAndTotals() {
        let s = freshStore()
        _ = s.record(id: "a", tier: .sprout, seconds: 90)
        _ = s.record(id: "b", tier: .oldGrowth, seconds: 300)
        #expect(s.totalCleared == 2)
        #expect(s.clearedCount(for: .sprout) == 1)
        #expect(s.clearedCount(for: .mycelium) == 0)
        #expect(s.bestSeconds(for: .mycelium) == nil)
    }

    @Test func persistsAcrossInstances() {
        let suite = "test-completions-persist"
        let d = UserDefaults(suiteName: suite)!
        d.removePersistentDomain(forName: suite)
        let a = CompletionStore(defaults: d)
        _ = a.record(id: "a", tier: .ancient, seconds: 120)
        let b = CompletionStore(defaults: d)
        #expect(b.isCleared("a"))
        #expect(b.bestSeconds(for: .ancient) == 120)
    }

    @Test func clearAllResets() {
        let s = freshStore()
        _ = s.record(id: "a", tier: .sprout, seconds: 90)
        s.clearAll()
        #expect(s.totalCleared == 0)
        #expect(s.hasAnyStats == false)
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter CompletionStoreTests`
Expected: FAIL — `cannot find 'CompletionStore' in scope`.

- [ ] **Step 3: Write the implementation**

```swift
// rootline/Daily/CompletionStore.swift
import Foundation

/// One cleared puzzle, keyed externally by its stable id.
struct Completion: Codable, Equatable, Sendable {
    let tier: Tier
    var bestSeconds: Int
}

/// Single source of truth for cleared status and derived stats. Keyed by the
/// puzzle's stable content id so history survives pool growth/reordering.
@MainActor
@Observable
final class CompletionStore {
    private static let key = "rootline_completions_v1"
    private let defaults: UserDefaults

    private(set) var byID: [String: Completion] = [:]

    init(defaults: UserDefaults = .standard) {
        self.defaults = defaults
        load()
    }

    func isCleared(_ id: String) -> Bool { byID[id] != nil }
    func isCleared(_ p: DailyPuzzle) -> Bool { isCleared(p.id) }

    /// Record a clear; keep the fastest time on re-clear. Returns true only when
    /// this beat a pre-existing per-tier best (the calm "your fastest yet"
    /// whisper). A first clear in a tier returns false.
    @discardableResult
    func record(id: String, tier: Tier, seconds: Int) -> Bool {
        let priorTierBest = bestSeconds(for: tier)
        if var existing = byID[id] {
            if seconds < existing.bestSeconds {
                existing.bestSeconds = seconds
                byID[id] = existing
            }
        } else {
            byID[id] = Completion(tier: tier, bestSeconds: seconds)
        }
        persist()
        if let prior = priorTierBest { return seconds < prior }
        return false
    }

    var totalCleared: Int { byID.count }
    func clearedCount(for tier: Tier) -> Int { byID.values.filter { $0.tier == tier }.count }
    func bestSeconds(for tier: Tier) -> Int? {
        byID.values.filter { $0.tier == tier }.map(\.bestSeconds).min()
    }
    var hasAnyStats: Bool { !byID.isEmpty }

    func clearAll() {
        byID = [:]
        persist()
    }

    private func load() {
        guard let data = defaults.data(forKey: Self.key),
              let decoded = try? JSONDecoder().decode([String: Completion].self, from: data) else { return }
        byID = decoded
    }
    private func persist() {
        if let data = try? JSONEncoder().encode(byID) {
            defaults.set(data, forKey: Self.key)
        }
    }
}
```

```bash
ln -s ../../../../../rootline/Daily/CompletionStore.swift \
  scripts/MycogridSolver/Sources/MycogridSolver/AppModel/CompletionStore.swift
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter CompletionStoreTests`
Expected: PASS (7 tests). Then run the full suite to confirm nothing regressed:
Run: `swift test --package-path scripts/MycogridSolver`
Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add rootline/Daily/CompletionStore.swift \
  scripts/MycogridSolver/Sources/MycogridSolver/AppModel/CompletionStore.swift \
  scripts/MycogridSolver/Tests/MycogridSolverTests/CompletionStoreTests.swift
git commit -m "feat: add id-keyed CompletionStore as completion/stats source of truth"
```

---

# Part 2 — Bundle generation + app/UI cutover (Xcode build + simulator)

Part 2 has no headless test target (the app has no XCTest target; SwiftUI work is GUI-bound). Each task ends with an **Xcode build + simulator verification** in place of an automated test. Build/run via the Xcode agent. The app must compile and run after every task — old types are removed only in the final cleanup task (Task 11), so intermediate states keep building.

Open the project: `rootline.xcodeproj` (scheme `rootline`, an iOS Simulator destination).

---

## Task 6: Generate `puzzles.json` and add it as a bundled resource

**Files:**
- Create: `rootline/Resources/puzzles.json` (generated, committed)
- Create: `rootline/Resources/puzzles.README.md` (records the exact reproduction commands)
- Modify: `rootline.xcodeproj/project.pbxproj` (add `puzzles.json` to the `rootline` target's Copy Bundle Resources — do this in Xcode, not by hand)

- [ ] **Step 1: Build the generator release binary**

```bash
cd /Users/michelleweirathmueller/dev/games/shroom-games/mycogrid
swift build -c release --package-path scripts/MycogridSolver --product mycogrid-generate
```
Expected: Build complete. Binary at `scripts/MycogridSolver/.build/release/mycogrid-generate`.

- [ ] **Step 2: Generate one file per tier (frequency-balanced counts)**

This may take a while on the larger tiers (pure-logic gating + dedup); progress logs go to stderr.

```bash
cd /Users/michelleweirathmueller/dev/games/shroom-games/mycogrid
BIN=scripts/MycogridSolver/.build/release/mycogrid-generate
mkdir -p /tmp/mycogrid-gen
$BIN --seed 1 --count 200 --tier sprout    --out /tmp/mycogrid-gen/sprout.json
$BIN --seed 1 --count 200 --tier mycelium  --out /tmp/mycogrid-gen/mycelium.json
$BIN --seed 1 --count 100 --tier ancient   --out /tmp/mycogrid-gen/ancient.json
$BIN --seed 1 --count 200 --tier oldGrowth --out /tmp/mycogrid-gen/oldGrowth.json
```
Expected: four files written; each prints `wrote N bytes`.

- [ ] **Step 3: Merge into one bundle**

```bash
cd /Users/michelleweirathmueller/dev/games/shroom-games/mycogrid
mkdir -p rootline/Resources
jq -s '{version: 1, tiers: (reduce .[] as $f ({}; . + $f.tiers))}' \
  /tmp/mycogrid-gen/sprout.json \
  /tmp/mycogrid-gen/mycelium.json \
  /tmp/mycogrid-gen/ancient.json \
  /tmp/mycogrid-gen/oldGrowth.json \
  > rootline/Resources/puzzles.json
```
Expected: `rootline/Resources/puzzles.json` exists with four tier keys.

- [ ] **Step 4: Validate the merged bundle (integrity guard)**

```bash
cd /Users/michelleweirathmueller/dev/games/shroom-games/mycogrid
swift run -c release --package-path scripts/MycogridSolver mycogrid-validate bundle rootline/Resources/puzzles.json
```
Expected: every row `PASS`, final line `700 entries, 0 failing`, exit code 0. (If any FAIL, stop — that is a real generator bug to surface, not suppress.)

Sanity-check the counts:
```bash
jq '.tiers | map_values(length)' rootline/Resources/puzzles.json
```
Expected: `{ "sprout": 200, "mycelium": 200, "ancient": 100, "oldGrowth": 200 }`.

- [ ] **Step 5: Record reproduction steps**

```markdown
<!-- rootline/Resources/puzzles.README.md -->
# puzzles.json — generated bundle

Do **not** hand-edit. The app's date→puzzle mapping depends on array order being
append-only: when extending, only *append* to a tier's array; never reorder or
remove entries.

Regenerate (drift guard — should reproduce byte-for-byte):

    BIN=scripts/MycogridSolver/.build/release/mycogrid-generate
    swift build -c release --package-path scripts/MycogridSolver --product mycogrid-generate
    $BIN --seed 1 --count 200 --tier sprout    --out /tmp/mycogrid-gen/sprout.json
    $BIN --seed 1 --count 200 --tier mycelium  --out /tmp/mycogrid-gen/mycelium.json
    $BIN --seed 1 --count 100 --tier ancient   --out /tmp/mycogrid-gen/ancient.json
    $BIN --seed 1 --count 200 --tier oldGrowth --out /tmp/mycogrid-gen/oldGrowth.json
    jq -s '{version:1, tiers:(reduce .[] as $f ({}; . + $f.tiers))}' \
      /tmp/mycogrid-gen/{sprout,mycelium,ancient,oldGrowth}.json > rootline/Resources/puzzles.json

Validate:

    swift run -c release --package-path scripts/MycogridSolver \
      mycogrid-validate bundle rootline/Resources/puzzles.json
```

- [ ] **Step 6: Add `puzzles.json` to the app target (Xcode)**

In Xcode: drag `rootline/Resources/puzzles.json` into the `rootline` group's `Resources`, **uncheck "Copy items"** (it's already in place), check the **rootline** target. Confirm it appears under Target `rootline` → Build Phases → Copy Bundle Resources.

- [ ] **Step 7: Build to confirm the resource bundles**

Xcode: Product → Build (⌘B). Expected: Build Succeeded, no errors.

- [ ] **Step 8: Commit**

```bash
git add rootline/Resources/puzzles.json rootline/Resources/puzzles.README.md rootline.xcodeproj/project.pbxproj
git commit -m "feat: generate + bundle puzzles.json (sprout/mycelium 200, ancient 100, oldGrowth 200)"
```

---

## Task 7: `AppState` daily wiring + Home "Today's grove"

Adds the daily flow additively: load `DailyService.live` + `CompletionStore`, add daily entry points, swap Home's Difficulty card for a Today card. Old `startGame`/`nextPuzzle`/`DifficultyView` stay in place (unused by Home now) so the app keeps compiling; they're removed in Task 11.

**Files:**
- Modify: `rootline/Views/RootView.swift` (AppState properties + methods; `.home` wiring; add `.archive` screen placeholder routing comes in Task 8)
- Modify: `rootline/Views/HomeView.swift` (replace Difficulty card with Today card; new params)

**Interfaces:**
- Consumes: `DailyService` (`live`, `puzzle(for:)`, `tier(for:)`), `CompletionStore` (`isCleared`, `record`), `DailyPuzzle`, `Board`.
- Produces (on `AppState`): `let daily: DailyService?`, `let completions: CompletionStore`, `var currentDate: Date`, `func startToday()`, `func startArchived(date: Date)`, `func recordClear(seconds: Int)`, plus a `todayContext` helper for Home.

- [ ] **Step 1: Add daily services + entry points to `AppState`**

In `rootline/Views/RootView.swift`, add to `AppState`'s stored properties (near `scoreStore`):

```swift
    let daily: DailyService? = DailyService.live()
    let completions: CompletionStore = CompletionStore()

    /// The calendar date of the board currently in play (today, or an archived day).
    private(set) var playingDate: Date = Date()
    private(set) var playingPuzzleID: String?
```

Add a live loader for `DailyService`. Create `rootline/Daily/DailyService+Live.swift`:

```swift
// rootline/Daily/DailyService+Live.swift
import Foundation

extension DailyService {
    /// Loads `puzzles.json` from the app bundle. Asserts in DEBUG on failure
    /// (a packaging defect); returns nil in release for a graceful surface.
    static func live(calendar: Calendar = .autoupdatingCurrent) -> DailyService? {
        guard let url = Bundle.main.url(forResource: "puzzles", withExtension: "json"),
              let data = try? Data(contentsOf: url) else {
            assertionFailure("puzzles.json missing from app bundle")
            return nil
        }
        do {
            return DailyService(bundle: try PuzzleBundle(data: data), calendar: calendar)
        } catch {
            assertionFailure("puzzles.json failed to decode: \(error)")
            return nil
        }
    }
}
```
(This file is app-only — do **not** symlink it into the package; `Bundle.main` resolves to the test runner there.)

Add the entry-point methods to `AppState`:

```swift
    /// Start today's daily puzzle.
    func startToday() {
        playToday(date: Date())
    }

    /// Start the puzzle for a specific calendar date (from the archive).
    func startArchived(date: Date) {
        playToday(date: date)
    }

    private func playToday(date: Date) {
        guard let daily, let dp = daily.puzzle(for: date) else { return }
        playingDate = date
        playingPuzzleID = dp.id
        activeBoard = Board(puzzle: dp.puzzle, tier: dp.tier, groveNumber: 0)
        saveProgress()
        screen = .play
    }

    /// Record a clear of the in-play puzzle. Returns true on a new per-tier best.
    @discardableResult
    func recordClear(seconds: Int) -> Bool {
        guard let id = playingPuzzleID, let tier = activeBoard?.tier else { return false }
        return completions.record(id: id, tier: tier, seconds: seconds)
    }
```

- [ ] **Step 2: Point Home's Play button at the daily flow**

In `RootView.content`, the `.home` case currently calls `appState.startGame(tier:)`. Replace its `HomeView(...)` construction with the new Today-based one:

```swift
        case .home:
            HomeView(
                today: appState.daily.flatMap { _ in appState.todayContext() },
                onPlayToday: { appState.startToday() },
                onArchive: { appState.openArchive() },     // added in Task 8; stub for now
                onStats: { appState.openStats() },
                onHowToPlay: { appState.startTutorial() },
                onSettings: { showSettings = true }
            )
            .transition(.opacity)
```

Add to `AppState` a `todayContext()` helper and a temporary `openArchive()` that no-ops until Task 8:

```swift
    struct TodayContext {
        let date: Date
        let tier: Tier
        let cleared: Bool
        let bestSeconds: Int?
    }

    func todayContext() -> TodayContext? {
        guard let daily, let dp = daily.puzzle(for: Date()) else { return nil }
        return TodayContext(
            date: Date(),
            tier: dp.tier,
            cleared: completions.isCleared(dp),
            bestSeconds: completions.bestSeconds(for: dp.tier)
        )
    }

    func openArchive() { /* wired in Task 8 */ }
```

- [ ] **Step 3: Rewrite `HomeView` with the Today card**

Replace `rootline/Views/HomeView.swift` body wholesale:

```swift
import SwiftUI
import ShroomKit

struct HomeView: View {
    let today: AppState.TodayContext?
    let onPlayToday: () -> Void
    let onArchive: () -> Void
    let onStats: () -> Void
    let onHowToPlay: () -> Void
    let onSettings: () -> Void

    @Environment(\.palette) private var palette

    var body: some View {
        VStack(spacing: 0) {
            HStack {
                Spacer()
                PillIconButton(systemName: "gearshape", accessibilityLabel: "Settings", action: onSettings)
            }
            Spacer(minLength: 0)
            VStack(spacing: 14) {
                RoundedRectangle(cornerRadius: 22, style: .continuous)
                    .fill(palette.pill)
                    .frame(width: 92, height: 92)
                    .overlay(MyceliumIcon().padding(16))
                Text("Mycogrid")
                    .font(.system(.largeTitle, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.text)
                Text("A cozy loop puzzle for mushroom foragers.")
                    .font(.system(.callout, design: .rounded))
                    .foregroundStyle(palette.sub)
                    .multilineTextAlignment(.center)
                    .frame(maxWidth: 230)
            }
            Spacer(minLength: 0)
            VStack(spacing: 11) {
                todayCard
                Button(today?.cleared == true ? "Play again" : "Play today", action: onPlayToday)
                    .buttonStyle(.shroomPrimary(prominent: true))
                    .disabled(today == nil)
                HStack(spacing: 6) {
                    textButton(icon: "calendar", title: "Archive", action: onArchive)
                    textButton(icon: "chart.bar.fill", title: "Stats", action: onStats)
                    textButton(icon: "questionmark.circle", title: "How to play", action: onHowToPlay)
                }
            }
            Spacer(minLength: 24)
        }
        .padding(.horizontal, 26)
        .padding(.vertical, 16)
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(palette.appBg.ignoresSafeArea())
    }

    private var todayCard: some View {
        VStack(alignment: .leading, spacing: 3) {
            EyebrowLabel("Today's grove")
            if let today {
                Text(today.date.formatted(.dateTime.weekday(.wide).month().day()))
                    .font(.system(.title3, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.text)
                HStack(spacing: 8) {
                    Text(today.tier.label)
                        .font(.system(.footnote, design: .rounded))
                        .foregroundStyle(palette.sub)
                    if today.cleared {
                        Text("· Cleared\(today.bestSeconds.map { " · \($0.asTimerString)" } ?? "")")
                            .font(.system(.footnote, design: .rounded).weight(.semibold))
                            .foregroundStyle(palette.accent)
                    }
                }
            } else {
                Text("Couldn't load today's puzzle")
                    .font(.system(.title3, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.text)
            }
        }
        .frame(maxWidth: .infinity, alignment: .leading)
        .padding(.horizontal, 18)
        .padding(.vertical, 15)
        .background(RoundedRectangle(cornerRadius: 18, style: .continuous).fill(palette.pill))
    }

    private func textButton(icon: String, title: String, action: @escaping () -> Void) -> some View {
        Button(action: action) {
            HStack(spacing: 8) {
                Image(systemName: icon)
                    .font(.system(.subheadline, design: .rounded).weight(.semibold))
                Text(title)
                    .font(.system(.subheadline, design: .rounded).weight(.semibold))
            }
            .foregroundStyle(palette.sub)
            .frame(maxWidth: .infinity)
            .frame(minHeight: 44)
            .padding(.horizontal, 6)
            .contentShape(Rectangle())
        }
        .buttonStyle(.plain)
    }
}
```

- [ ] **Step 4: Verify (Xcode build + simulator)**

Build & run (⌘R) on a simulator. Expected:
- Home shows "Today's grove" with today's weekday/date and the correct tier for the weekday (per the day→tier table — verify against the real current day).
- Tapping **Play today** opens a board of the right size.
- Solving it: the win card still appears (recording is wired in Task 9; for now it may still call the old `scoreStore` path — acceptable, app compiles).
- No Difficulty card on Home.

- [ ] **Step 5: Commit**

```bash
git add rootline/Views/RootView.swift rootline/Views/HomeView.swift rootline/Daily/DailyService+Live.swift
git commit -m "feat: wire daily services into AppState and Home Today's grove card"
```

---

## Task 8: `ArchiveView` — the completion calendar

**Files:**
- Create: `rootline/Views/ArchiveView.swift`
- Modify: `rootline/Views/RootView.swift` (add `.archive` to `Screen`, `openArchive()`, archive-floor persistence in `Settings`, routing, and `onPlayArchived`)
- Modify: `rootline/Storage/Settings.swift` (persist the archive floor; remove `tier` is deferred to Task 11)

**Interfaces:**
- Consumes: `DailyService` (`archiveDates`, `puzzle(for:)`, `currentStreak`, `archiveFloor`), `CompletionStore` (`isCleared`), `Tier`.
- Produces: `ArchiveView` and `AppState.openArchive()` / `startArchived(date:)` (the latter from Task 7).

- [ ] **Step 1: Persist the archive floor in `Settings`**

Add to `rootline/Storage/Settings.swift` — a new key in `Keys` and a lazily-fixed floor:

```swift
        static let archiveFloor = "rootline_archive_floor_v1"
```
and a property:

```swift
    /// The archive's back-scroll floor: fixed once at first access to
    /// (firstOpen − 14 days), then never moved forward.
    var archiveFloor: Date {
        if let t = UserDefaults.standard.object(forKey: Keys.archiveFloor) as? Double {
            return Date(timeIntervalSince1970: t)
        }
        let floor = DailyService.archiveFloor(firstOpen: Date())
        UserDefaults.standard.set(floor.timeIntervalSince1970, forKey: Keys.archiveFloor)
        return floor
    }
```

- [ ] **Step 2: Add the `.archive` screen + routing in `RootView`**

In `enum Screen`, add `case archive`. In `AppState`, replace the Task-7 stub:

```swift
    func openArchive() { screen = .archive }
```
Add the `.archive` case to `RootView.content`:

```swift
        case .archive:
            ArchiveView(
                daily: appState.daily,
                completions: appState.completions,
                floor: appState.settings.archiveFloor,
                onPlay: { date in appState.startArchived(date: date) },
                onClose: { appState.goHome() }
            )
            .transition(.opacity)
```

- [ ] **Step 3: Write `ArchiveView`**

```swift
// rootline/Views/ArchiveView.swift
import SwiftUI
import ShroomKit

struct ArchiveView: View {
    let daily: DailyService?
    @Bindable var completions: CompletionStore
    let floor: Date
    let onPlay: (Date) -> Void
    let onClose: () -> Void

    @Environment(\.palette) private var palette

    private var dates: [Date] {
        daily?.archiveDates(floor: floor, today: Date()) ?? []
    }
    private var streak: Int {
        guard let daily else { return 0 }
        return daily.currentStreak(today: Date()) { completions.isCleared($0) }
    }

    private let columns = Array(repeating: GridItem(.flexible(), spacing: 8), count: 4)

    var body: some View {
        VStack(spacing: 0) {
            ScreenHeader("Archive", onBack: onClose)
                .padding(.horizontal, 22)
                .padding(.top, 12)
                .padding(.bottom, 8)
            if streak > 0 {
                Text(streak == 1 ? "1 day streak" : "\(streak) day streak")
                    .font(.system(.footnote, design: .rounded).weight(.medium))
                    .foregroundStyle(palette.sub)
                    .padding(.bottom, 10)
            }
            ScrollView {
                LazyVGrid(columns: columns, spacing: 8) {
                    ForEach(dates, id: \.timeIntervalSince1970) { date in
                        cell(for: date)
                    }
                }
                .padding(.horizontal, 22)
                .padding(.bottom, 30)
            }
            .scrollIndicators(.hidden)
        }
        .background(palette.appBg.ignoresSafeArea())
    }

    @ViewBuilder
    private func cell(for date: Date) -> some View {
        let dp = daily?.puzzle(for: date)
        let cleared = dp.map { completions.isCleared($0) } ?? false
        let isToday = Calendar.autoupdatingCurrent.isDateInToday(date)
        Button {
            onPlay(date)
        } label: {
            VStack(spacing: 4) {
                Text(date.formatted(.dateTime.day()))
                    .font(.system(.subheadline, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.text)
                Image(systemName: cleared ? "checkmark.circle.fill" : "circle")
                    .font(.system(.footnote))
                    .foregroundStyle(cleared ? palette.accent : palette.sub.opacity(0.4))
            }
            .frame(maxWidth: .infinity)
            .frame(height: 60)
            .background(
                RoundedRectangle(cornerRadius: 12, style: .continuous)
                    .fill(tint(for: dp?.tier))
            )
            .overlay(
                RoundedRectangle(cornerRadius: 12, style: .continuous)
                    .strokeBorder(palette.accent, lineWidth: isToday ? 2 : 0)
            )
            .contentShape(Rectangle())
        }
        .buttonStyle(.plain)
        .accessibilityLabel(Text("\(date.formatted(.dateTime.weekday(.wide).month().day())), \(dp?.tier.label ?? ""), \(cleared ? "cleared" : "not cleared")"))
    }

    /// Subtle per-tier tint so the weekly size rhythm reads at a glance.
    private func tint(for tier: Tier?) -> Color {
        switch tier {
        case .sprout:    return palette.pill
        case .mycelium:  return palette.pill.opacity(0.85)
        case .ancient:   return palette.tierSelBg.opacity(0.6)
        case .oldGrowth: return palette.tierSelBg
        case nil:        return palette.pill
        }
    }
}
```

Note: confirm `palette.tierSelBg` exists (used by `WinCard`). If a different calm tint reads better, adjust during visual review — this is one of the spec's open UI questions.

- [ ] **Step 4: Verify (Xcode build + simulator)**

Build & run. Expected:
- Home → Archive shows a grid of recent days, most-recent (today, ringed) first, reaching ~2 weeks back (floor).
- Each cell shows the day number, a cleared/uncleared mark, and a subtle tier tint.
- Tapping a past day opens that day's puzzle at the size matching its weekday.
- Clearing today then returning to Archive shows today's cell checked, and the streak line appears.

- [ ] **Step 5: Commit**

```bash
git add rootline/Views/ArchiveView.swift rootline/Views/RootView.swift rootline/Storage/Settings.swift
git commit -m "feat: add Archive completion calendar with backfillable streak"
```

---

## Task 9: Play/win cutover — record by id, drop "Next puzzle"

**Files:**
- Modify: `rootline/Views/RootView.swift` (`.play` wiring: pass `completions`, drop `onNext`)
- Modify: `rootline/Views/PlayView.swift` (record via `recordClear`; remove `onNext`; header label)
- Modify: `rootline/Views/WinCard.swift` (remove "Next puzzle"; "Back to archive"/"Done")
- Modify: `rootline/Views/RevealedCard.swift` (remove "Next puzzle")

**Interfaces:**
- Consumes: `AppState.recordClear(seconds:)`, `CompletionStore`.
- Produces: a play screen that records completions by id and has no next-puzzle treadmill.

- [ ] **Step 1: Rewire the `.play` case in `RootView`**

Replace the `.play` `PlayView(...)` construction:

```swift
        case .play:
            if let board = appState.activeBoard {
                PlayView(
                    board: board,
                    settings: appState.settings,
                    onRecordClear: { seconds in appState.recordClear(seconds: seconds) },
                    onBack: { appState.goHome() },
                    onMenu: {
                        appState.clearSavedProgress()
                        appState.goHome()
                    },
                    onSave: { appState.saveProgress() },
                    onClearProgress: { appState.clearSavedProgress() }
                )
                .transition(.opacity)
            }
```
(Back now returns Home; archived days are re-reachable from the Archive. Removing the difficulty round-trip.)

- [ ] **Step 2: Update `PlayView`**

In `rootline/Views/PlayView.swift`:
- Replace the `scoreStore` property and `onNext` with:

```swift
    let onRecordClear: (Int) -> Bool
```
(remove `@Bindable var scoreStore: ScoreStore` and `let onNext: () -> Void`.)

- In the `.overlay(alignment: .bottom)`, change the `WinCard`/`RevealedCard` calls to drop `onNext` (use the new signatures from Steps 3–4): `WinCard(board: board, fastestYet: fastestYet, onMenu: onMenu)` and `RevealedCard(board: board, onMenu: onMenu)`.

- In `.onChange(of: board.solveTick)`, replace the `scoreStore.record(...)` block:

```swift
        .onChange(of: board.solveTick) { _, _ in
            guard board.isSolved else { return }
            onClearProgress()
            fastestYet = onRecordClear(board.elapsedSeconds)
        }
```

- In `header`, the title currently reads `Text("Grove #\(board.groveNumber)")`. Replace with the tier label as the title (groveNumber is gone):

```swift
            VStack(spacing: 1) {
                EyebrowLabel(board.tier?.label ?? "Lesson")
                Text(board.tier.map { "\($0.cols)×\($0.rows)" } ?? "Lesson")
                    .font(.system(.title3, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.text)
            }
```

- [ ] **Step 3: Update `WinCard`**

```swift
// rootline/Views/WinCard.swift — replace the ResultCard call:
struct WinCard: View {
    let board: Board
    let fastestYet: Bool
    let onMenu: () -> Void

    @Environment(\.palette) private var palette

    var body: some View {
        ResultCard(
            title: "Network connected!",
            subtitle: subtitle,
            note: fastestYet ? "Your fastest yet" : nil,
            primaryLabel: "Done", onPrimary: onMenu,
            secondaryLabel: nil, onSecondary: nil
        ) {
            RoundedRectangle(cornerRadius: Radius.md, style: .continuous)
                .fill(palette.tierSelBg)
                .frame(width: 44, height: 44)
                .overlay(
                    Image(systemName: "checkmark")
                        .font(.system(.title3, design: .rounded).weight(.bold))
                        .foregroundStyle(palette.accent)
                )
        }
    }

    private var subtitle: String {
        let tierLabel = board.tier?.label ?? "Lesson"
        let size = "\(board.puzzle.cols)×\(board.puzzle.rows)"
        let time = board.elapsedSeconds.asTimerString
        return "\(tierLabel) · \(size) · cleared in \(time)"
    }
}
```
Confirm `ResultCard` accepts a `nil` `secondaryLabel`/`onSecondary`. If its signature requires non-optional values, instead pass a single primary action and check `ResultCard`'s API in ShroomKit; adapt to a one-button variant. (ShroomKit `ResultCard` — verify the optional-secondary support before editing; if absent, use the primary-only initializer.)

- [ ] **Step 4: Update `RevealedCard`** the same way — remove `onNext`, make "Done" the primary calling `onMenu`, drop the secondary.

- [ ] **Step 5: Verify (Xcode build + simulator)**

Build & run. Expected:
- Play today → solve → win card shows time, **no "Next puzzle"**, a single "Done" returns Home.
- Re-open Home: today's card shows "Cleared · m:ss".
- Open Archive: today checked; a faster re-clear of an earlier-tier day shows the "Your fastest yet" whisper appropriately.
- "Show solution" → RevealedCard has no "Next puzzle".

- [ ] **Step 6: Commit**

```bash
git add rootline/Views/PlayView.swift rootline/Views/WinCard.swift rootline/Views/RevealedCard.swift rootline/Views/RootView.swift
git commit -m "feat: record clears by id; remove next-puzzle cycling from play/win"
```

---

## Task 10: `StatsView` derives from `CompletionStore`

**Files:**
- Modify: `rootline/Views/StatsView.swift` (read `CompletionStore` instead of `ScoreStore`)
- Modify: `rootline/Views/RootView.swift` (`.stats` passes `completions`)

**Interfaces:**
- Consumes: `CompletionStore` (`stat`-equivalents: `bestSeconds(for:)`, `clearedCount(for:)`, `totalCleared`, `hasAnyStats`, `clearAll`).

- [ ] **Step 1: Update the `.stats` route**

In `RootView.content`:

```swift
        case .stats:
            StatsView(
                completions: appState.completions,
                onClose: { appState.goHome() }
            )
            .transition(.opacity)
```

- [ ] **Step 2: Update `StatsView`**

Replace `@Bindable var scoreStore: ScoreStore` with `@Bindable var completions: CompletionStore`. Update the body's three reads:

- `if scoreStore.hasAnyStats` → `if completions.hasAnyStats`
- `"\(scoreStore.totalCleared) puzzles cleared"` → `"\(completions.totalCleared) groves cleared"`
- `scoreStore.clearAll()` → `completions.clearAll()`

Replace `tierCard` to read per-tier values from `completions`:

```swift
    private func tierCard(_ tier: Tier) -> some View {
        let best = completions.bestSeconds(for: tier)
        let count = completions.clearedCount(for: tier)
        return VStack(alignment: .leading, spacing: 6) {
            HStack(alignment: .firstTextBaseline) {
                EyebrowLabel(tier.label)
                Spacer()
                Text(tier.shortMeta)
                    .font(.system(.caption2, design: .rounded))
                    .foregroundStyle(palette.sub)
            }
            if let best {
                Text("Your fastest: \(best.asTimerString)")
                    .font(.system(.body, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.text)
                    .monospacedDigit()
                Text("\(count) cleared")
                    .font(.system(.footnote, design: .rounded))
                    .foregroundStyle(palette.sub)
            } else {
                Text("Not cleared yet")
                    .font(.system(.footnote, design: .rounded))
                    .foregroundStyle(palette.sub)
            }
        }
        .frame(maxWidth: .infinity, alignment: .leading)
        .padding(.horizontal, 16)
        .padding(.vertical, 14)
        .background(RoundedRectangle(cornerRadius: 14, style: .continuous).fill(palette.pill))
    }
```
Also update the clear-alert copy: "This wipes every fastest time and cleared count across all tiers. Can't be undone." stays accurate.

- [ ] **Step 3: Verify (Xcode build + simulator)**

Build & run. Clear a couple of puzzles across two tiers, open Stats. Expected: per-tier fastest + count reflect the completions; total reads "N groves cleared"; Clear wipes them.

- [ ] **Step 4: Commit**

```bash
git add rootline/Views/StatsView.swift rootline/Views/RootView.swift
git commit -m "feat: derive StatsView from CompletionStore"
```

---

## Task 11: Cleanup — retire old model + re-key resume + retire pool audit

Now remove everything the daily model supersedes, in one task so the build breaks only briefly and is fixed within it.

**Files:**
- Modify: `rootline/Model/PuzzleData.swift` (remove per-tier lists + `puzzles(for:)`; keep `lessons`/`Lesson`)
- Modify: `rootline/Model/Board.swift` (`init?(restoring:)` via `DailyService`; `groveNumber` → drop or default)
- Modify: `rootline/Storage/ProgressStore.swift` (re-key snapshot to puzzle id + date; bump to `_v2`)
- Modify: `rootline/Views/RootView.swift` (remove `startGame`/`nextPuzzle`/`currentPuzzleIndex`; `finishLaunch` restore via id)
- Modify: `rootline/Storage/Settings.swift` (remove `tier` property + key)
- Delete: `rootline/Views/DifficultyView.swift`
- Delete: `rootline/Storage/ScoreStore.swift`
- Modify: `scripts/MycogridSolver/Sources/MycogridSolver/Pool.swift` (remove `auditPool()`; keep `auditPuzzle`/`PoolReport`)
- Modify: `scripts/MycogridSolver/Sources/mycogrid-validate/main.swift` (remove the `pool` case)
- Delete: `scripts/MycogridSolver/Tests/MycogridSolverTests/PoolTests.swift`

**Interfaces:**
- `PuzzleProgress` becomes `{ puzzleID: String, playedDate: Date, activeEdges, xEdges, mode, elapsedSeconds, hintsUsed }`.
- `Board.init?(restoring:using:)` takes the `DailyService` to rebuild the puzzle from `puzzleID`'s date.

- [ ] **Step 1: Re-key `ProgressStore` + snapshot**

`rootline/Storage/ProgressStore.swift`:

```swift
import Foundation

struct PuzzleProgress: Codable, Sendable {
    let puzzleID: String
    let playedDate: Date
    let tier: Tier
    let activeEdges: [Edge]
    let xEdges: [Edge]
    let mode: DrawMode
    let elapsedSeconds: Int
    let hintsUsed: Int
}

@MainActor
final class ProgressStore {
    private static let key = "rootline_in_progress_v2"

    func load() -> PuzzleProgress? {
        guard let data = UserDefaults.standard.data(forKey: Self.key) else { return nil }
        return try? JSONDecoder().decode(PuzzleProgress.self, from: data)
    }
    func save(_ progress: PuzzleProgress) {
        guard let data = try? JSONEncoder().encode(progress) else { return }
        UserDefaults.standard.set(data, forKey: Self.key)
    }
    func clear() { UserDefaults.standard.removeObject(forKey: Self.key) }
}
```

- [ ] **Step 2: Update `Board` snapshot + restore**

In `rootline/Model/Board.swift`:
- Remove `let groveNumber: Int` and the `groveNumber` init params (replace usages). `Board.init(puzzle:tier:groveNumber:allowHints:)` → `Board(puzzle:tier:allowHints:)`. Update the two call sites in `AppState.playToday` (drop `groveNumber: 0`).
- Add the in-play identity needed for snapshot. Since `AppState` already holds `playingPuzzleID`/`playingDate`, have `snapshot()` take them in. Replace `snapshot()`:

```swift
    func snapshot(puzzleID: String, playedDate: Date) -> PuzzleProgress? {
        guard let tier else { return nil }
        return PuzzleProgress(
            puzzleID: puzzleID,
            playedDate: playedDate,
            tier: tier,
            activeEdges: Array(activeEdges),
            xEdges: Array(xEdges),
            mode: mode,
            elapsedSeconds: elapsedSeconds,
            hintsUsed: hintsUsed
        )
    }
```
- Replace `convenience init?(restoring:)` to rebuild via `DailyService`:

```swift
    convenience init?(restoring progress: PuzzleProgress, using daily: DailyService) {
        guard let dp = daily.puzzle(for: progress.playedDate), dp.id == progress.puzzleID else { return nil }
        self.init(puzzle: dp.puzzle, tier: dp.tier, allowHints: true)
        self.activeEdges = Set(progress.activeEdges)
        self.xEdges = Set(progress.xEdges)
        self.mode = progress.mode
        self.elapsedSeconds = progress.elapsedSeconds
        self.hintsUsed = progress.hintsUsed
        if model.isSolved(active: activeEdges) { isSolved = true }
    }
```

- [ ] **Step 3: Update `AppState`**

In `rootline/Views/RootView.swift`:
- Remove `private var currentPuzzleIndex`, `func startGame(tier:)`, `func nextPuzzle()`.
- `saveProgress()` uses the new snapshot signature:

```swift
    func saveProgress() {
        guard let board = activeBoard, !board.isSolved, let id = playingPuzzleID,
              let snapshot = board.snapshot(puzzleID: id, playedDate: playingDate) else { return }
        progressStore.save(snapshot)
    }
```
- `finishLaunch()` restore branch:

```swift
        if let saved = progressStore.load(), let daily,
           let board = Board(restoring: saved, using: daily), !board.isSolved {
            activeBoard = board
            playingDate = saved.playedDate
            playingPuzzleID = saved.puzzleID
            screen = .play
            return
        }
```

- [ ] **Step 4: Trim `PuzzleData` + `Settings` + delete dead views/stores**

- `rootline/Model/PuzzleData.swift`: delete the `static let sprout/mycelium/ancient/oldGrowth` arrays and `static func puzzles(for:)`. Keep `Lesson` and `lessons`. (The package symlink to `PuzzleData.swift` stays valid — only contents shrank.)
- `rootline/Storage/Settings.swift`: remove the `tier` computed property and `Keys.tier` and the `self.tier = ...` line in `init`.
- Delete `rootline/Views/DifficultyView.swift` and `rootline/Storage/ScoreStore.swift`. In Xcode, remove their file references from the target.

- [ ] **Step 5: Retire the legacy pool audit in the package**

- `scripts/MycogridSolver/Sources/MycogridSolver/Pool.swift`: delete `auditPool()` (it referenced the removed `PuzzleData.puzzles(for:)`). Keep `PoolReport` and `auditPuzzle` (used by `auditBundle`).
- `scripts/MycogridSolver/Sources/mycogrid-validate/main.swift`: delete the `case "pool":` block; update usage string to `usage: mycogrid-validate <bundle <path> | file <path>>`.
- Delete `scripts/MycogridSolver/Tests/MycogridSolverTests/PoolTests.swift`.

- [ ] **Step 6: Verify package**

```bash
swift test --package-path scripts/MycogridSolver
```
Expected: builds; all remaining tests PASS (no `PoolTests`).

- [ ] **Step 7: Verify app (Xcode build + simulator) — full regression**

Build & run. Expected:
- Home (no Difficulty card), Today's grove, Play today, win/Done, Archive (browse + play past days + streak), Stats (derived), Settings/tutorial intact.
- Background mid-puzzle, relaunch → resumes the same in-progress board (id-keyed restore).
- No references to grove numbers anywhere.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "refactor: retire grove-index model, ScoreStore, DifficultyView, and legacy pool audit"
```

---

## Self-Review

**Spec coverage:**
- Load bundle as resource → Task 6. Decode bridge → Task 1.
- Date→puzzle deterministic, append-only, id-keyed cleared status → Tasks 3, 5; append-only stability test in Task 3.
- Day→tier weekly curve → Task 3 (`tier(for:)`), verified Task 7.
- "Today's grove" on Home → Task 7. Difficulty picker removed → Tasks 7 (unlinked) + 11 (deleted).
- Archive calendar + per-install floor + tint + tap-to-play → Task 8; floor persistence → Task 8 Step 1.
- Backfillable streak → Task 4, surfaced Task 8.
- Fresh-start single CompletionStore; ScoreStore retired; ProgressStore re-keyed → Tasks 5, 10, 11.
- Remove next-puzzle cycling → Task 9.
- Integrity guard (`mycogrid-validate bundle`) → Task 2, run in Task 6.
- Bundle counts 200/200/100/200 → Task 6.
- Error handling (missing/corrupt bundle asserts in DEBUG, graceful nil in release; empty pool → nil) → `DailyService.live` (Task 7), `puzzle(for:)` empty-pool test (Task 3).

**Placeholder scan:** No "TBD"/"add error handling"/"similar to". Two flagged verifications are explicit, not placeholders: confirm `ResultCard` supports a nil secondary (Task 9 Step 3) and confirm `palette.tierSelBg` tint reads calm (Task 8 Step 3) — both name the exact check and fallback.

**Type consistency:** `DailyPuzzle{id,tier,puzzle}`, `PuzzleBundle.puzzles(for:)`, `DailyService.puzzle(for:)`/`tier(for:)`/`tierOccurrenceIndex(for:tier:)`/`archiveDates(floor:today:)`/`currentStreak(today:isCleared:)`/`archiveFloor(firstOpen:calendar:)`, `CompletionStore.record(id:tier:seconds:)->Bool`/`isCleared`/`bestSeconds(for:)`/`clearedCount(for:)`/`totalCleared`, `AppState.startToday()`/`startArchived(date:)`/`recordClear(seconds:)`/`todayContext()`, `Board(puzzle:tier:allowHints:)`/`snapshot(puzzleID:playedDate:)`/`init?(restoring:using:)`, `PuzzleProgress{puzzleID,playedDate,tier,…}` — used consistently across tasks.

**Known coupling note for the executor:** Task 11 is intentionally the only build-breaking-then-fixing task; do it atomically. Part 1 (Tasks 1–5) is fully headless and independent of Part 2.
