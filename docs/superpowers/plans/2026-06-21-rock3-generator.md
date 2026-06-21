# Mycogrid Generator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an offline `mycogrid-generate` CLI that emits a validated JSON bundle of uniquely-solvable, pure-logic puzzles per tier.

**Architecture:** A new executable target inside the existing `MycogridSolver` SwiftPM package, backed by library code that reuses `Solver`, `PuzzleModel`, `Edge`, and `Cell`. The pipeline grows a random simply-connected region (cell accretion), gates it on the fully-clued board, greedily hides clues while the 2-rule solver still cracks it with `guesses == 0`, and serializes the result deterministically.

**Tech Stack:** Swift 5.9, SwiftPM, XCTest (the package's existing test framework — *not* Swift Testing), Foundation (for `Data`/`JSONEncoder`/`String(format:)`).

## Global Constraints

- **Package path:** all commands run from the worktree root `rootline-rock3/`; the package lives at `scripts/MycogridSolver/`. Use `--package-path scripts/MycogridSolver` so no `cd` is needed.
- **Test framework:** XCTest (`import XCTest`, `final class … : XCTestCase`, `func test_…`). Match the existing files in `Tests/MycogridSolverTests/`.
- **Module access:** `Puzzle`, `Tier`, `PuzzleModel`, `SplitMix64`, region/hider/pipeline helpers stay **internal**; tests reach them via `@testable import MycogridSolver`. Only the CLI entry points (`GenerateOptions`, `GenerateError`, `generateBundleData`) are `public`. This keeps `Puzzle` internal, matching how `mycogrid-validate` only touches public `solve`/`loadClues`.
- **Determinism is load-bearing:** never iterate a `Set` or `Dictionary` and feed it to `randomElement`/`shuffled` — Swift's `Set`/`Dictionary` iteration order is not stable across runs. Always sort to an `Array` by `($0.r, $0.c)` first, then apply the seeded RNG.
- **Pure-logic gate (verbatim rule):** a puzzle is admitted only when `solve(...).verdict == .unique` **and** `.trace.guesses == 0`.
- **Terminology:** never write "Slitherlink" in code or comments. Product is Mycogrid; repo is rootline.
- **Tiers (fixed sizes):** `sprout` 4×6, `mycelium` 5×7, `ancient` 6×9, `oldGrowth` 7×10 (from `Tier.swift`; do not redefine).
- **Commit trailer:** end every commit message with `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`.

---

## File Structure

New library files (all in `scripts/MycogridSolver/Sources/MycogridSolver/`):
- `RNG.swift` — `SplitMix64` seeded PRNG.
- `Region.swift` — `isSimplyConnected(...)` predicate + `RegionGenerator` (accretion).
- `Identity.swift` — `canonicalKey(...)`, `fnv1a64(...)`, `puzzleID(...)`.
- `ClueHider.swift` — greedy clue-hiding.
- `Generator.swift` — `GeneratedPuzzle`, `generateOne`, `generateBundle`, and the public CLI entry (`GenerateOptions`/`GenerateError`/`generateBundleData`).
- `BundleJSON.swift` — Codable output types + deterministic `encodeBundle(...)`.

New CLI target:
- `scripts/MycogridSolver/Sources/mycogrid-generate/main.swift` — arg parsing + file write.

New test files (all in `scripts/MycogridSolver/Tests/MycogridSolverTests/`):
- `RNGTests.swift`, `RegionTests.swift`, `IdentityTests.swift`, `ClueHiderTests.swift`, `GeneratorTests.swift`, `BundleJSONTests.swift`.

Modified:
- `scripts/MycogridSolver/Package.swift` — add the `mycogrid-generate` executable target.

---

### Task 1: Seeded PRNG (SplitMix64)

**Files:**
- Create: `scripts/MycogridSolver/Sources/MycogridSolver/RNG.swift`
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/RNGTests.swift`

**Interfaces:**
- Produces: `struct SplitMix64: RandomNumberGenerator { init(seed: UInt64); mutating func next() -> UInt64 }` (internal). Conforming to `RandomNumberGenerator` lets every later task use `Int.random(in:using:)`, `Double.random(in:using:)`, `Array.shuffled(using:)`, `Array.randomElement(using:)` reproducibly.

- [ ] **Step 1: Write the failing test**

```swift
// RNGTests.swift
import XCTest
@testable import MycogridSolver

final class RNGTests: XCTestCase {
    func test_sameSeed_producesSameSequence() {
        var a = SplitMix64(seed: 42)
        var b = SplitMix64(seed: 42)
        let seqA = (0..<5).map { _ in a.next() }
        let seqB = (0..<5).map { _ in b.next() }
        XCTAssertEqual(seqA, seqB)
    }

    func test_differentSeed_producesDifferentSequence() {
        var a = SplitMix64(seed: 1)
        var b = SplitMix64(seed: 2)
        XCTAssertNotEqual(a.next(), b.next())
    }

    func test_drivesStdlibRandomDeterministically() {
        var a = SplitMix64(seed: 7)
        var b = SplitMix64(seed: 7)
        let xa = Int.random(in: 0..<1000, using: &a)
        let xb = Int.random(in: 0..<1000, using: &b)
        XCTAssertEqual(xa, xb)
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter RNGTests`
Expected: FAIL — `cannot find 'SplitMix64' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// RNG.swift
/// Reproducible PRNG (SplitMix64). Unlike `SystemRandomNumberGenerator`,
/// the same seed always yields the same sequence — required for byte-identical
/// generated bundles.
struct SplitMix64: RandomNumberGenerator {
    private var state: UInt64
    init(seed: UInt64) { self.state = seed }

    mutating func next() -> UInt64 {
        state &+= 0x9E37_79B9_7F4A_7C15
        var z = state
        z = (z ^ (z >> 30)) &* 0xBF58_476D_1CE4_E5B9
        z = (z ^ (z >> 27)) &* 0x94D0_49BB_1331_11EB
        return z ^ (z >> 31)
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter RNGTests`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add scripts/MycogridSolver/Sources/MycogridSolver/RNG.swift scripts/MycogridSolver/Tests/MycogridSolverTests/RNGTests.swift
git commit -m "$(printf 'feat: add SplitMix64 seeded PRNG for the generator\n\nCo-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>')"
```

---

### Task 2: Simply-connected region predicate

**Files:**
- Create: `scripts/MycogridSolver/Sources/MycogridSolver/Region.swift`
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/RegionTests.swift`

**Interfaces:**
- Consumes: `Cell` (from `PuzzleData.swift`, has `.c`, `.r`, `init(c:r:)`).
- Produces: `func isSimplyConnected(_ inside: Set<Cell>, cols: Int, rows: Int) -> Bool` (internal) — true iff the region is non-empty, smaller than the whole grid, edge-connected, and hole-free. Also `func fourNeighbors(_ cell: Cell) -> [Cell]` (internal helper, reused by Task 3).

- [ ] **Step 1: Write the failing test**

```swift
// RegionTests.swift
import XCTest
@testable import MycogridSolver

final class RegionTests: XCTestCase {
    private func cells(_ pairs: [[Int]]) -> Set<Cell> {
        Set(pairs.map { Cell(c: $0[0], r: $0[1]) })
    }

    func test_solidRectangle_isSimplyConnected() {
        // 2x2 block inside a 3x3 grid.
        let region = cells([[0,0],[1,0],[0,1],[1,1]])
        XCTAssertTrue(isSimplyConnected(region, cols: 3, rows: 3))
    }

    func test_regionWithHole_isRejected() {
        // Ring of 8 cells around an empty center in a 3x3 grid — center (1,1) is a hole.
        let ring = cells([[0,0],[1,0],[2,0],[0,1],[2,1],[0,2],[1,2],[2,2]])
        XCTAssertFalse(isSimplyConnected(ring, cols: 3, rows: 3))
    }

    func test_disconnectedRegion_isRejected() {
        // Two separate cells in a 3x3 grid.
        let region = cells([[0,0],[2,2]])
        XCTAssertFalse(isSimplyConnected(region, cols: 3, rows: 3))
    }

    func test_emptyRegion_isRejected() {
        XCTAssertFalse(isSimplyConnected([], cols: 3, rows: 3))
    }

    func test_wholeGrid_isRejected() {
        let all = cells([[0,0],[1,0],[0,1],[1,1]])
        XCTAssertFalse(isSimplyConnected(all, cols: 2, rows: 2))
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter RegionTests`
Expected: FAIL — `cannot find 'isSimplyConnected' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// Region.swift
import Foundation

/// The four edge-adjacent neighbors of a cell (may be out of bounds; callers filter).
func fourNeighbors(_ cell: Cell) -> [Cell] {
    [
        Cell(c: cell.c - 1, r: cell.r),
        Cell(c: cell.c + 1, r: cell.r),
        Cell(c: cell.c, r: cell.r - 1),
        Cell(c: cell.c, r: cell.r + 1),
    ]
}

/// True iff `inside` is non-empty, smaller than the whole grid, edge-connected,
/// and hole-free. A hole would enclose "outside" cells the loop can't reach,
/// producing two loops instead of one.
func isSimplyConnected(_ inside: Set<Cell>, cols: Int, rows: Int) -> Bool {
    guard !inside.isEmpty, inside.count < cols * rows else { return false }

    // 1. Edge-connected: flood the region from any member.
    var seen: Set<Cell> = []
    var stack = [inside.first!]
    seen.insert(inside.first!)
    while let cur = stack.popLast() {
        for n in fourNeighbors(cur) where inside.contains(n) && !seen.contains(n) {
            seen.insert(n)
            stack.append(n)
        }
    }
    if seen.count != inside.count { return false }

    // 2. Hole-free: flood the "sea" from a padded corner just outside the grid.
    //    Every in-bounds non-inside cell must be reachable; any unreached one is a hole.
    func inPadded(_ c: Cell) -> Bool {
        c.c >= -1 && c.c <= cols && c.r >= -1 && c.r <= rows
    }
    var sea: Set<Cell> = [Cell(c: -1, r: -1)]
    var stack2 = [Cell(c: -1, r: -1)]
    while let cur = stack2.popLast() {
        for n in fourNeighbors(cur)
        where inPadded(n) && !inside.contains(n) && !sea.contains(n) {
            sea.insert(n)
            stack2.append(n)
        }
    }
    let reachedInBounds = sea.filter { $0.c >= 0 && $0.c < cols && $0.r >= 0 && $0.r < rows }.count
    return reachedInBounds == cols * rows - inside.count
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter RegionTests`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add scripts/MycogridSolver/Sources/MycogridSolver/Region.swift scripts/MycogridSolver/Tests/MycogridSolverTests/RegionTests.swift
git commit -m "$(printf 'feat: add simply-connected region predicate (hole + connectivity check)\n\nCo-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>')"
```

---

### Task 3: Region accretion generator

**Files:**
- Modify: `scripts/MycogridSolver/Sources/MycogridSolver/Region.swift` (append `RegionGenerator`)
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/RegionTests.swift` (append cases)

**Interfaces:**
- Consumes: `SplitMix64` (Task 1), `isSimplyConnected`/`fourNeighbors` (Task 2).
- Produces: `struct RegionGenerator { init(cols: Int, rows: Int); func generate(using rng: inout some RandomNumberGenerator) -> Set<Cell> }` (internal). The returned region is connected and non-trivial but **may contain a hole** — the caller (Task 6) re-checks with `isSimplyConnected` and retries on failure.

- [ ] **Step 1: Write the failing test**

```swift
// Append to RegionTests.swift
extension RegionTests {
    func test_generate_isDeterministicForSameSeed() {
        var a = SplitMix64(seed: 99)
        var b = SplitMix64(seed: 99)
        let ra = RegionGenerator(cols: 4, rows: 6).generate(using: &a)
        let rb = RegionGenerator(cols: 4, rows: 6).generate(using: &b)
        XCTAssertEqual(ra, rb)
    }

    func test_generate_isNonTrivialAndConnected() {
        var rng = SplitMix64(seed: 3)
        let gen = RegionGenerator(cols: 5, rows: 7)
        // Run a handful of seeds; every region is connected and within the fill band.
        for s in 0..<20 {
            var r = SplitMix64(seed: UInt64(s))
            let region = gen.generate(using: &r)
            XCTAssertFalse(region.isEmpty)
            XCTAssertLessThan(region.count, 5 * 7)
            // Connectivity holds even though holes may exist.
            XCTAssertTrue(regionIsConnected(region))
        }
        _ = rng
    }

    // Local connectivity helper (does not check holes).
    private func regionIsConnected(_ inside: Set<Cell>) -> Bool {
        guard let first = inside.first else { return false }
        var seen: Set<Cell> = [first]
        var stack = [first]
        while let cur = stack.popLast() {
            for n in fourNeighbors(cur) where inside.contains(n) && !seen.contains(n) {
                seen.insert(n); stack.append(n)
            }
        }
        return seen.count == inside.count
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter RegionTests`
Expected: FAIL — `cannot find 'RegionGenerator' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// Append to Region.swift

/// Grows a random connected region by cell accretion. Result is connected and
/// sized within ~40–60% of the grid, but may contain a hole; callers validate
/// with `isSimplyConnected` and retry.
struct RegionGenerator {
    let cols: Int
    let rows: Int

    func generate(using rng: inout some RandomNumberGenerator) -> Set<Cell> {
        let total = cols * rows
        let frac = Double.random(in: 0.4...0.6, using: &rng)
        let target = max(1, min(total - 1, Int((Double(total) * frac).rounded())))

        let seed = Cell(c: Int.random(in: 0..<cols, using: &rng),
                        r: Int.random(in: 0..<rows, using: &rng))
        var inside: Set<Cell> = [seed]
        var frontier: Set<Cell> = Set(inBoundsNeighbors(seed))

        while inside.count < target && !frontier.isEmpty {
            // Sort frontier to an array before randomElement — Set order is not stable.
            let ordered = frontier.sorted { ($0.r, $0.c) < ($1.r, $1.c) }
            let pick = ordered.randomElement(using: &rng)!
            inside.insert(pick)
            frontier.remove(pick)
            for n in inBoundsNeighbors(pick) where !inside.contains(n) {
                frontier.insert(n)
            }
        }
        return inside
    }

    private func inBoundsNeighbors(_ cell: Cell) -> [Cell] {
        fourNeighbors(cell).filter { $0.c >= 0 && $0.c < cols && $0.r >= 0 && $0.r < rows }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter RegionTests`
Expected: PASS (8 tests total).

- [ ] **Step 5: Commit**

```bash
git add scripts/MycogridSolver/Sources/MycogridSolver/Region.swift scripts/MycogridSolver/Tests/MycogridSolverTests/RegionTests.swift
git commit -m "$(printf 'feat: add cell-accretion region generator\n\nCo-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>')"
```

---

### Task 4: Puzzle identity (canonical key + stable id)

**Files:**
- Create: `scripts/MycogridSolver/Sources/MycogridSolver/Identity.swift`
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/IdentityTests.swift`

**Interfaces:**
- Consumes: `Cell`.
- Produces (all internal):
  - `func canonicalKey(cols: Int, rows: Int, inside: Set<Cell>) -> String` — exact, order-independent string; used for dedup.
  - `func fnv1a64(_ s: String) -> UInt64` — stable hash (Swift's `Hasher` is randomly seeded per run, so it cannot be used here).
  - `func puzzleID(cols: Int, rows: Int, inside: Set<Cell>) -> String` — 12-char hex id derived from `canonicalKey`. (Resolves the spec's open question: 12 hex chars = 48 bits, collision-safe across the low-thousands pool while dedup still uses the exact `canonicalKey`.)

- [ ] **Step 1: Write the failing test**

```swift
// IdentityTests.swift
import XCTest
@testable import MycogridSolver

final class IdentityTests: XCTestCase {
    private func cells(_ pairs: [[Int]]) -> Set<Cell> {
        Set(pairs.map { Cell(c: $0[0], r: $0[1]) })
    }

    func test_canonicalKey_isOrderIndependent() {
        let a = cells([[0,0],[1,0],[1,1]])
        let b = cells([[1,1],[0,0],[1,0]])
        XCTAssertEqual(canonicalKey(cols: 2, rows: 2, inside: a),
                       canonicalKey(cols: 2, rows: 2, inside: b))
    }

    func test_id_isStableForSameRegion() {
        let region = cells([[0,0],[1,0],[1,1]])
        XCTAssertEqual(puzzleID(cols: 2, rows: 2, inside: region),
                       puzzleID(cols: 2, rows: 2, inside: region))
    }

    func test_id_differsForDifferentRegion() {
        let a = cells([[0,0],[1,0]])
        let b = cells([[0,0],[0,1]])
        XCTAssertNotEqual(puzzleID(cols: 2, rows: 2, inside: a),
                          puzzleID(cols: 2, rows: 2, inside: b))
    }

    func test_id_is12HexChars() {
        let region = cells([[0,0]])
        let id = puzzleID(cols: 1, rows: 1, inside: region)
        XCTAssertEqual(id.count, 12)
        XCTAssertTrue(id.allSatisfy { $0.isHexDigit })
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter IdentityTests`
Expected: FAIL — `cannot find 'canonicalKey' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// Identity.swift
import Foundation

/// Order-independent string identity for a region. Cells sorted (row, col).
func canonicalKey(cols: Int, rows: Int, inside: Set<Cell>) -> String {
    let sorted = inside.sorted { ($0.r, $0.c) < ($1.r, $1.c) }
    return "\(cols)x\(rows):" + sorted.map { "\($0.c),\($0.r)" }.joined(separator: ";")
}

/// FNV-1a 64-bit. Stable across processes (unlike the stdlib `Hasher`).
func fnv1a64(_ s: String) -> UInt64 {
    var hash: UInt64 = 0xcbf2_9ce4_8422_2325
    for byte in s.utf8 {
        hash ^= UInt64(byte)
        hash = hash &* 0x0000_0100_0000_01b3
    }
    return hash
}

/// 12-char hex id derived from the canonical region key.
func puzzleID(cols: Int, rows: Int, inside: Set<Cell>) -> String {
    let h = fnv1a64(canonicalKey(cols: cols, rows: rows, inside: inside))
    return String(String(format: "%016llx", h).prefix(12))
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter IdentityTests`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add scripts/MycogridSolver/Sources/MycogridSolver/Identity.swift scripts/MycogridSolver/Tests/MycogridSolverTests/IdentityTests.swift
git commit -m "$(printf 'feat: add content-derived stable puzzle id + canonical region key\n\nCo-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>')"
```

---

### Task 5: Greedy clue-hider

**Files:**
- Create: `scripts/MycogridSolver/Sources/MycogridSolver/ClueHider.swift`
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/ClueHiderTests.swift`

**Interfaces:**
- Consumes: `PuzzleModel` (has `.puzzle` with `.cols`/`.rows`, and `.clues: [Cell: Int]`), `solve`, `PuzzleClues`, `SplitMix64`, the pure-logic gate rule.
- Produces: `struct ClueHider { let model: PuzzleModel; func hide(using rng: inout some RandomNumberGenerator) -> Set<Cell> }` (internal). Returns the set of cells whose clues are hidden; the remaining visible clues keep the puzzle `.unique` with `guesses == 0`, and the hiding is **maximal** (no further single clue can be hidden).

- [ ] **Step 1: Write the failing test**

```swift
// ClueHiderTests.swift
import XCTest
@testable import MycogridSolver

final class ClueHiderTests: XCTestCase {
    // A small, fully-clued, pure-logic-unique base puzzle (Sprout grove #1 shape).
    private func sproutModel() -> PuzzleModel {
        PuzzleModel(Puzzle(cols: 4, rows: 6, inside: [
            [1,0],[2,0],
            [0,1],[1,1],[2,1],[3,1],
            [0,2],[1,2],[2,2],[3,2],
            [0,3],[1,3],[2,3],[3,3],
            [0,4],[1,4],[2,4],[3,4],
            [1,5],[2,5]
        ]))
    }

    private func solveVisible(_ model: PuzzleModel, hidden: Set<Cell>) -> SolveResult {
        var visible: [Cell: Int] = [:]
        for (cell, n) in model.clues where !hidden.contains(cell) { visible[cell] = n }
        return solve(PuzzleClues(cols: model.puzzle.cols, rows: model.puzzle.rows, clues: visible))
    }

    func test_hide_keepsPureLogicUniqueness() {
        let model = sproutModel()
        var rng = SplitMix64(seed: 5)
        let hidden = ClueHider(model: model).hide(using: &rng)
        let result = solveVisible(model, hidden: hidden)
        XCTAssertEqual(result.verdict, .unique)
        XCTAssertEqual(result.trace.guesses, 0)
    }

    func test_hide_isMaximal() {
        let model = sproutModel()
        var rng = SplitMix64(seed: 5)
        let hidden = ClueHider(model: model).hide(using: &rng)
        // Every still-shown clue, if additionally hidden, must break pure-logic uniqueness.
        for cell in model.clues.keys where !hidden.contains(cell) {
            var more = hidden; more.insert(cell)
            let result = solveVisible(model, hidden: more)
            let stillGood = result.verdict == .unique && result.trace.guesses == 0
            XCTAssertFalse(stillGood, "clue at (\(cell.c),\(cell.r)) could still be hidden — not maximal")
        }
    }

    func test_hide_isDeterministic() {
        let model = sproutModel()
        var a = SplitMix64(seed: 5)
        var b = SplitMix64(seed: 5)
        XCTAssertEqual(ClueHider(model: model).hide(using: &a),
                       ClueHider(model: model).hide(using: &b))
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter ClueHiderTests`
Expected: FAIL — `cannot find 'ClueHider' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// ClueHider.swift

/// Greedily hides as many clues as the 2-rule solver tolerates while keeping the
/// puzzle uniquely solvable by pure logic (`guesses == 0`). The achievable
/// density emerges from what the solver can deduce.
struct ClueHider {
    let model: PuzzleModel

    func hide(using rng: inout some RandomNumberGenerator) -> Set<Cell> {
        let cols = model.puzzle.cols
        let rows = model.puzzle.rows
        var hidden: Set<Cell> = []

        // Deterministic order: sort clue cells, then shuffle with the seeded RNG.
        let order = model.clues.keys
            .sorted { ($0.r, $0.c) < ($1.r, $1.c) }
            .shuffled(using: &rng)

        for cell in order {
            var trial = hidden
            trial.insert(cell)
            var visible: [Cell: Int] = [:]
            for (cl, n) in model.clues where !trial.contains(cl) { visible[cl] = n }
            let result = solve(PuzzleClues(cols: cols, rows: rows, clues: visible))
            if result.verdict == .unique && result.trace.guesses == 0 {
                hidden = trial
            }
        }
        return hidden
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter ClueHiderTests`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add scripts/MycogridSolver/Sources/MycogridSolver/ClueHider.swift scripts/MycogridSolver/Tests/MycogridSolverTests/ClueHiderTests.swift
git commit -m "$(printf 'feat: add greedy pure-logic clue-hider\n\nCo-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>')"
```

---

### Task 6: Generation pipeline + driver

**Files:**
- Create: `scripts/MycogridSolver/Sources/MycogridSolver/Generator.swift`
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/GeneratorTests.swift`

**Interfaces:**
- Consumes: `RegionGenerator`, `isSimplyConnected` (Task 2/3); `Puzzle`, `PuzzleModel`, `solve`, `PuzzleClues`, `Verdict`, `Rule` (existing); `ClueHider` (Task 5); `puzzleID`, `canonicalKey` (Task 4); `SplitMix64` (Task 1); `Tier` (existing, has `.cols`, `.rows`, `.label`, `.rawValue`, `Tier.allCases`, `Tier(rawValue:)`).
- Produces (internal):
  - `struct GeneratedPuzzle: Sendable { let cols: Int; let rows: Int; let inside: [Cell]; let hideClues: [Cell]; let id: String; let shownClueCount: Int; let rulesFired: [Rule: Int] }` — `inside`/`hideClues` are sorted `(row, col)`.
  - `func generateOne(tier: Tier, using rng: inout SplitMix64) -> GeneratedPuzzle?` — one pipeline pass; `nil` if the region has a hole or the fully-clued board fails the pure-logic gate.
  - `func generateBundle(tiers: [Tier], countPerTier: Int, seed: UInt64, progress: (String) -> Void) -> [Tier: [GeneratedPuzzle]]` — loops per tier, dedups by `canonicalKey`, caps attempts.
- The public CLI entry (`GenerateOptions`/`GenerateError`/`generateBundleData`) is added in Task 8 to this same file; do not add it yet.

- [ ] **Step 1: Write the failing test**

```swift
// GeneratorTests.swift
import XCTest
@testable import MycogridSolver

final class GeneratorTests: XCTestCase {
    // Re-solve a generated puzzle from only its visible clues and assert the
    // core invariant: unique, pure-logic, and matching the derived solution.
    private func assertCoreInvariant(_ gp: GeneratedPuzzle) {
        let puzzle = Puzzle(
            cols: gp.cols, rows: gp.rows,
            inside: gp.inside.map { [$0.c, $0.r] },
            hide: gp.hideClues.map { [$0.c, $0.r] }
        )
        let model = PuzzleModel(puzzle)
        var visible: [Cell: Int] = [:]
        for (cell, n) in model.clues where !Set(gp.hideClues).contains(cell) { visible[cell] = n }
        let result = solve(PuzzleClues(cols: gp.cols, rows: gp.rows, clues: visible))
        XCTAssertEqual(result.verdict, .unique)
        XCTAssertEqual(result.trace.guesses, 0)
        XCTAssertEqual(result.solution, model.solution)
    }

    func test_generateBundle_everyPuzzleSatisfiesCoreInvariant() {
        let bundle = generateBundle(tiers: [.sprout], countPerTier: 3, seed: 1, progress: { _ in })
        let puzzles = bundle[.sprout] ?? []
        XCTAssertEqual(puzzles.count, 3)
        for gp in puzzles { assertCoreInvariant(gp) }
    }

    func test_generateBundle_dedupsByRegion() {
        let bundle = generateBundle(tiers: [.sprout], countPerTier: 5, seed: 2, progress: { _ in })
        let puzzles = bundle[.sprout] ?? []
        let keys = puzzles.map { canonicalKey(cols: $0.cols, rows: $0.rows, inside: Set($0.inside)) }
        XCTAssertEqual(Set(keys).count, keys.count, "duplicate regions in bundle")
    }

    func test_generateBundle_isDeterministic() {
        let a = generateBundle(tiers: [.sprout], countPerTier: 3, seed: 7, progress: { _ in })
        let b = generateBundle(tiers: [.sprout], countPerTier: 3, seed: 7, progress: { _ in })
        let ka = (a[.sprout] ?? []).map { $0.id }
        let kb = (b[.sprout] ?? []).map { $0.id }
        XCTAssertEqual(ka, kb)
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter GeneratorTests`
Expected: FAIL — `cannot find 'generateBundle' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// Generator.swift
import Foundation

struct GeneratedPuzzle: Sendable {
    let cols: Int
    let rows: Int
    let inside: [Cell]      // sorted (row, col)
    let hideClues: [Cell]   // sorted (row, col)
    let id: String
    let shownClueCount: Int
    let rulesFired: [Rule: Int]
}

/// One pipeline pass: region → gate on fully-clued board → greedy hide → record.
/// Returns nil if the region has a hole or the full board isn't pure-logic unique.
func generateOne(tier: Tier, using rng: inout SplitMix64) -> GeneratedPuzzle? {
    let inside = RegionGenerator(cols: tier.cols, rows: tier.rows).generate(using: &rng)
    guard isSimplyConnected(inside, cols: tier.cols, rows: tier.rows) else { return nil }

    let puzzle = Puzzle(cols: tier.cols, rows: tier.rows, inside: inside.map { [$0.c, $0.r] })
    let model = PuzzleModel(puzzle)

    // Gate: a region whose fully-clued board needs guessing can never be pure-logic.
    let full = solve(PuzzleClues(cols: tier.cols, rows: tier.rows, clues: model.clues))
    guard full.verdict == .unique, full.trace.guesses == 0 else { return nil }

    let hidden = ClueHider(model: model).hide(using: &rng)

    var visible: [Cell: Int] = [:]
    for (cell, n) in model.clues where !hidden.contains(cell) { visible[cell] = n }
    let finalResult = solve(PuzzleClues(cols: tier.cols, rows: tier.rows, clues: visible))

    let sortedCells: (Set<Cell>) -> [Cell] = { $0.sorted { ($0.r, $0.c) < ($1.r, $1.c) } }
    return GeneratedPuzzle(
        cols: tier.cols,
        rows: tier.rows,
        inside: sortedCells(inside),
        hideClues: sortedCells(hidden),
        id: puzzleID(cols: tier.cols, rows: tier.rows, inside: inside),
        shownClueCount: model.clues.count - hidden.count,
        rulesFired: finalResult.trace.rulesFired
    )
}

/// Drives `generateOne` per tier until `countPerTier` unique puzzles are collected,
/// deduping by region. Caps attempts so an exhausted tier can't loop forever.
func generateBundle(
    tiers: [Tier],
    countPerTier: Int,
    seed: UInt64,
    progress: (String) -> Void
) -> [Tier: [GeneratedPuzzle]] {
    var rng = SplitMix64(seed: seed)
    var out: [Tier: [GeneratedPuzzle]] = [:]

    for tier in tiers {
        var results: [GeneratedPuzzle] = []
        var seenKeys: Set<String> = []
        var attempts = 0
        let maxAttempts = max(1000, countPerTier * 200)

        while results.count < countPerTier && attempts < maxAttempts {
            attempts += 1
            guard let gp = generateOne(tier: tier, using: &rng) else { continue }
            let key = canonicalKey(cols: gp.cols, rows: gp.rows, inside: Set(gp.inside))
            if seenKeys.contains(key) { continue }
            seenKeys.insert(key)
            results.append(gp)
            if results.count % 10 == 0 {
                progress("\(tier.label): \(results.count)/\(countPerTier)")
            }
        }
        if results.count < countPerTier {
            progress("\(tier.label): WARNING only \(results.count)/\(countPerTier) after \(attempts) attempts")
        } else {
            progress("\(tier.label): done \(results.count)/\(countPerTier) (\(attempts) attempts)")
        }
        out[tier] = results
    }
    return out
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter GeneratorTests`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add scripts/MycogridSolver/Sources/MycogridSolver/Generator.swift scripts/MycogridSolver/Tests/MycogridSolverTests/GeneratorTests.swift
git commit -m "$(printf 'feat: add generation pipeline + per-tier driver with dedup\n\nCo-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>')"
```

---

### Task 7: Deterministic JSON bundle encoding

**Files:**
- Create: `scripts/MycogridSolver/Sources/MycogridSolver/BundleJSON.swift`
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/BundleJSONTests.swift`

**Interfaces:**
- Consumes: `GeneratedPuzzle`, `Tier` (`.rawValue`), `Rule` (Task 6 + existing).
- Produces (internal): `func encodeBundle(_ bundle: [Tier: [GeneratedPuzzle]], seed: UInt64) throws -> Data` — emits the `{version, tiers}` JSON with `.sortedKeys` so the same input yields byte-identical output. Also the Codable shapes `BundleJSON`, `PuzzleEntryJSON`, `MetaJSON`.

- [ ] **Step 1: Write the failing test**

```swift
// BundleJSONTests.swift
import XCTest
@testable import MycogridSolver

final class BundleJSONTests: XCTestCase {
    func test_encode_isByteIdenticalForSameBundle() throws {
        let bundle = generateBundle(tiers: [.sprout], countPerTier: 2, seed: 4, progress: { _ in })
        let a = try encodeBundle(bundle, seed: 4)
        let b = try encodeBundle(bundle, seed: 4)
        XCTAssertEqual(a, b)
    }

    func test_encode_decodesToExpectedStructure() throws {
        let bundle = generateBundle(tiers: [.sprout], countPerTier: 2, seed: 4, progress: { _ in })
        let data = try encodeBundle(bundle, seed: 4)
        let decoded = try JSONDecoder().decode(BundleJSON.self, from: data)
        XCTAssertEqual(decoded.version, 1)
        let sprout = try XCTUnwrap(decoded.tiers["sprout"])
        XCTAssertEqual(sprout.count, 2)
        let first = sprout[0]
        XCTAssertEqual(first.id.count, 12)
        XCTAssertFalse(first.inside.isEmpty)
        XCTAssertEqual(first.meta.seed, 4)
        XCTAssertGreaterThanOrEqual(first.meta.shownClueCount, 0)
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter BundleJSONTests`
Expected: FAIL — `cannot find 'encodeBundle' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// BundleJSON.swift
import Foundation

struct BundleJSON: Codable {
    let version: Int
    let tiers: [String: [PuzzleEntryJSON]]
}

struct PuzzleEntryJSON: Codable {
    let id: String
    let cols: Int
    let rows: Int
    let inside: [[Int]]
    let hideClues: [[Int]]
    let meta: MetaJSON
}

struct MetaJSON: Codable {
    let shownClueCount: Int
    let rulesFired: [String: Int]
    let seed: UInt64
}

/// Serializes a generated bundle. `.sortedKeys` + already-sorted cell arrays make
/// the output byte-identical for identical input (regenerate-and-diff drift guard).
func encodeBundle(_ bundle: [Tier: [GeneratedPuzzle]], seed: UInt64) throws -> Data {
    var tiers: [String: [PuzzleEntryJSON]] = [:]
    for (tier, puzzles) in bundle {
        tiers[tier.rawValue] = puzzles.map { gp in
            PuzzleEntryJSON(
                id: gp.id,
                cols: gp.cols,
                rows: gp.rows,
                inside: gp.inside.map { [$0.c, $0.r] },
                hideClues: gp.hideClues.map { [$0.c, $0.r] },
                meta: MetaJSON(
                    shownClueCount: gp.shownClueCount,
                    rulesFired: ["clue": gp.rulesFired[.clue] ?? 0,
                                 "dot": gp.rulesFired[.dot] ?? 0],
                    seed: seed
                )
            )
        }
    }
    let encoder = JSONEncoder()
    encoder.outputFormatting = [.sortedKeys, .prettyPrinted]
    return try encoder.encode(BundleJSON(version: 1, tiers: tiers))
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test --package-path scripts/MycogridSolver --filter BundleJSONTests`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add scripts/MycogridSolver/Sources/MycogridSolver/BundleJSON.swift scripts/MycogridSolver/Tests/MycogridSolverTests/BundleJSONTests.swift
git commit -m "$(printf 'feat: add deterministic JSON bundle encoding\n\nCo-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>')"
```

---

### Task 8: Public CLI entry point + arg parsing

**Files:**
- Modify: `scripts/MycogridSolver/Sources/MycogridSolver/Generator.swift` (append public API)
- Create: `scripts/MycogridSolver/Sources/mycogrid-generate/main.swift`
- Modify: `scripts/MycogridSolver/Package.swift`
- Test: `scripts/MycogridSolver/Tests/MycogridSolverTests/GeneratorTests.swift` (append)

**Interfaces:**
- Consumes: `generateBundle`, `encodeBundle`, `Tier(rawValue:)`, `Tier.allCases`.
- Produces (public):
  - `struct GenerateOptions { var tierNames: [String]?; var count: Int; var seed: UInt64; init(tierNames: [String]?, count: Int, seed: UInt64) }`
  - `enum GenerateError: Error, Equatable { case unknownTier(String) }`
  - `func generateBundleData(_ options: GenerateOptions, progress: (String) -> Void = { _ in }) throws -> Data`

- [ ] **Step 1: Write the failing test**

```swift
// Append to GeneratorTests.swift
extension GeneratorTests {
    func test_generateBundleData_unknownTier_throws() {
        let opts = GenerateOptions(tierNames: ["bogus"], count: 1, seed: 1)
        XCTAssertThrowsError(try generateBundleData(opts)) { error in
            XCTAssertEqual(error as? GenerateError, .unknownTier("bogus"))
        }
    }

    func test_generateBundleData_isDeterministic() throws {
        let opts = GenerateOptions(tierNames: ["sprout"], count: 2, seed: 11)
        let a = try generateBundleData(opts)
        let b = try generateBundleData(opts)
        XCTAssertEqual(a, b)
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test --package-path scripts/MycogridSolver --filter GeneratorTests`
Expected: FAIL — `cannot find 'GenerateOptions' in scope`.

- [ ] **Step 3: Write minimal implementation**

Append to `Generator.swift`:

```swift
// MARK: - Public CLI entry

public struct GenerateOptions {
    public var tierNames: [String]?   // nil = all tiers
    public var count: Int
    public var seed: UInt64
    public init(tierNames: [String]?, count: Int, seed: UInt64) {
        self.tierNames = tierNames
        self.count = count
        self.seed = seed
    }
}

public enum GenerateError: Error, Equatable {
    case unknownTier(String)
}

/// Resolves tier names, runs the driver, and returns the encoded JSON bundle.
public func generateBundleData(
    _ options: GenerateOptions,
    progress: (String) -> Void = { _ in }
) throws -> Data {
    let tiers: [Tier]
    if let names = options.tierNames {
        tiers = try names.map { name in
            guard let t = Tier(rawValue: name) else { throw GenerateError.unknownTier(name) }
            return t
        }
    } else {
        tiers = Tier.allCases
    }
    let bundle = generateBundle(
        tiers: tiers,
        countPerTier: options.count,
        seed: options.seed,
        progress: progress
    )
    return try encodeBundle(bundle, seed: options.seed)
}
```

Create `mycogrid-generate/main.swift`:

```swift
import Foundation
import MycogridSolver

func fail(_ msg: String, code: Int32) -> Never {
    FileHandle.standardError.write(Data((msg + "\n").utf8))
    exit(code)
}

// Parse --seed <int> --count <int> [--tier <name>] --out <path>
var seed: UInt64?
var count: Int?
var tier: String?
var out: String?

var i = 1
let argv = CommandLine.arguments
while i < argv.count {
    switch argv[i] {
    case "--seed":  i += 1; seed = i < argv.count ? UInt64(argv[i]) : nil
    case "--count": i += 1; count = i < argv.count ? Int(argv[i]) : nil
    case "--tier":  i += 1; tier = i < argv.count ? argv[i] : nil
    case "--out":   i += 1; out = i < argv.count ? argv[i] : nil
    default: fail("unknown argument: \(argv[i])", code: 2)
    }
    i += 1
}

guard let seed, let count, let out else {
    fail("usage: mycogrid-generate --seed <int> --count <int> [--tier <name>] --out <path>", code: 2)
}

let options = GenerateOptions(tierNames: tier.map { [$0] }, count: count, seed: seed)
do {
    let data = try generateBundleData(options) { line in
        FileHandle.standardError.write(Data((line + "\n").utf8))
    }
    try data.write(to: URL(fileURLWithPath: out))
    print("wrote \(data.count) bytes to \(out)")
} catch let GenerateError.unknownTier(name) {
    fail("unknown tier: \(name) (valid: sprout, mycelium, ancient, oldGrowth)", code: 2)
} catch {
    fail("error: \(error)", code: 1)
}
```

Modify `Package.swift` — add the executable target:

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "MycogridSolver",
    targets: [
        .target(name: "MycogridSolver"),
        .executableTarget(name: "mycogrid-validate", dependencies: ["MycogridSolver"]),
        .executableTarget(name: "mycogrid-generate", dependencies: ["MycogridSolver"]),
        .testTarget(name: "MycogridSolverTests", dependencies: ["MycogridSolver"]),
    ]
)
```

- [ ] **Step 4: Run tests + build the CLI**

Run: `swift test --package-path scripts/MycogridSolver --filter GeneratorTests`
Expected: PASS (5 tests total in GeneratorTests).

Run: `swift build --package-path scripts/MycogridSolver`
Expected: `Build complete!` with no errors.

- [ ] **Step 5: Commit**

```bash
git add scripts/MycogridSolver/Sources/MycogridSolver/Generator.swift scripts/MycogridSolver/Sources/mycogrid-generate/main.swift scripts/MycogridSolver/Package.swift scripts/MycogridSolver/Tests/MycogridSolverTests/GeneratorTests.swift
git commit -m "$(printf 'feat: add mycogrid-generate CLI target\n\nCo-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>')"
```

---

### Task 9: End-to-end smoke test + self-audit via mycogrid-validate

**Files:**
- (No new source) — integration check using both CLIs.

**Interfaces:**
- Consumes: the built `mycogrid-generate` and existing `mycogrid-validate file <path>` CLIs.

This task has no unit test; it confirms the whole pipeline runs and that the
already-trusted validator independently agrees with the generator's gate.

- [ ] **Step 1: Generate a small real bundle**

Run:
```bash
swift run --package-path scripts/MycogridSolver mycogrid-generate --seed 1 --count 5 --tier sprout --out /tmp/mycogrid-sprout.json
```
Expected: progress lines on stderr, then `wrote <N> bytes to /tmp/mycogrid-sprout.json`.

- [ ] **Step 2: Confirm the bundle is well-formed and deterministic**

Run:
```bash
swift run --package-path scripts/MycogridSolver mycogrid-generate --seed 1 --count 5 --tier sprout --out /tmp/mycogrid-sprout2.json
diff /tmp/mycogrid-sprout.json /tmp/mycogrid-sprout2.json && echo "DETERMINISTIC OK"
```
Expected: `DETERMINISTIC OK` (no diff).

- [ ] **Step 3: Self-audit each puzzle through mycogrid-validate**

The bundle groups puzzles by tier, but `mycogrid-validate file` expects a single
`{cols, rows, clues:[{c,r,n}]}` payload (see `JSONInput.swift`). Extract each
puzzle's *visible* clues and validate. Run this check:

```bash
python3 - <<'PY'
import json, subprocess, tempfile, os
bundle = json.load(open("/tmp/mycogrid-sprout.json"))
def derived_clues(cols, rows, inside):
    inside = {(c, r) for c, r in inside}
    def isin(c, r): return (c, r) in inside
    # count solution edges around each cell (mirror of PuzzleModel)
    clues = {}
    for r in range(rows):
        for c in range(cols):
            n = 0
            # top/bottom/left/right edge present iff it separates inside/outside
            if isin(c, r) != isin(c, r-1): n += 1
            if isin(c, r) != isin(c, r+1): n += 1
            if isin(c, r) != isin(c-1, r): n += 1
            if isin(c, r) != isin(c+1, r): n += 1
            clues[(c, r)] = n
    return clues
ok = True
for tier, puzzles in bundle["tiers"].items():
    for p in puzzles:
        clues = derived_clues(p["cols"], p["rows"], p["inside"])
        hide = {(c, r) for c, r in p["hideClues"]}
        visible = [{"c": c, "r": r, "n": n} for (c, r), n in clues.items() if (c, r) not in hide]
        payload = {"cols": p["cols"], "rows": p["rows"], "clues": visible}
        with tempfile.NamedTemporaryFile("w", suffix=".json", delete=False) as f:
            json.dump(payload, f); path = f.name
        r = subprocess.run(
            ["swift", "run", "--package-path", "scripts/MycogridSolver",
             "mycogrid-validate", "file", path],
            capture_output=True, text=True)
        os.unlink(path)
        if r.returncode != 0:
            ok = False
            print(f"FAIL {tier} {p['id']}:\n{r.stdout}{r.stderr}")
print("ALL UNIQUE OK" if ok else "VALIDATION FAILURES — see above")
PY
```
Expected: `ALL UNIQUE OK`. If any puzzle fails, that is a real generator bug — fix it (do not weaken the validator), then re-run from Task 6.

- [ ] **Step 4: Clean up temp files**

Run: `rm -f /tmp/mycogrid-sprout.json /tmp/mycogrid-sprout2.json`

- [ ] **Step 5: No commit** (verification only; nothing to add).

---

## Self-Review

**Spec coverage:**
- New `mycogrid-generate` CLI in `MycogridSolver` package → Task 8.
- Reuses `Solver`/`PuzzleModel`/`Edge`/`Cell` → Tasks 5, 6 (no rule logic duplicated).
- Pure-logic gate (unique + guesses==0) → Tasks 5, 6, asserted in 6 & 9.
- Reproducible (seeded PRNG, byte-identical) → Tasks 1, 6, 7, 8, 9 step 2.
- Cell-accretion region gen + no-hole guard → Tasks 2, 3, 6.
- Density emerges (greedy hide to cap) → Task 5.
- Any valid region (no shape heuristics) → Task 3 (none added).
- ~100–200/tier fixed bundle as app resource → driver `countPerTier` (Task 6); CLI `--count` (Task 8). Quantity is a run-time arg, not hardcoded — matches spec.
- Output format {version, tiers, id, cols, rows, inside, hideClues, meta} → Task 7.
- Content-derived stable id → Task 4.
- Self-audit via mycogrid-validate → Task 9.
- Testing invariants (region props, hole detection, hiding correctness/maximality, core invariant, determinism, identity) → Tasks 2,3,4,5,6,7.

**Open questions resolved in the plan:**
- Fill fraction: fixed `0.4...0.6` band for all tiers (Task 3).
- Id length: 12 hex chars (Task 4).
- Dedup: exact `canonicalKey` only; reflections/rotations count as distinct (Task 6).

**Placeholder scan:** none — every code step shows complete code.

**Type consistency:** `GeneratedPuzzle` fields, `generateOne`/`generateBundle`/`encodeBundle`/`generateBundleData` signatures, and `GenerateOptions`/`GenerateError` are used identically across Tasks 6–9. `puzzleID` returns 12 chars (Task 4) and is asserted at 12 in Tasks 4 and 7. The pure-logic gate (`verdict == .unique && trace.guesses == 0`) is written identically in Tasks 5, 6, and the Task 9 audit.

**Note on a known constraint:** the under-clued performance limit (solver commit `5022c32`) means large/sparse tiers (Ancient, Old Growth) validate slowly. All automated tests use `.sprout` with small counts to stay fast; the full ~100–200/tier production run is an offline batch the operator runs by hand, with stderr progress so a slow tier is visible rather than appearing hung.
