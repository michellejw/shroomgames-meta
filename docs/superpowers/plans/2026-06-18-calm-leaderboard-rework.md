# Calm Leaderboard Rework — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Rootline's arcade-style leaderboard (3-letter initials, ranked top-5, "New record!" entry sheet) with a quiet per-tier stats pattern: silent best-time + completion-count tracking, a calm "Stats" screen, and a win-card whisper only when you beat your own best.

**Architecture:** `ScoreStore` is rewritten from a ranked list of `ScoreEntry` rows into a per-tier `TierStat` (best seconds + cleared count), persisted to `UserDefaults`. Solving a puzzle calls `record(seconds:for:)`, which returns a `ClearOutcome` driving an optional win-card whisper. The old `WinEntrySheet` and ranked `BestTimesView` are removed; a new `StatsView` shows one line per tier. `ScoreStore` gains an injectable `UserDefaults` so its pure logic can be unit-tested in isolation.

**Tech Stack:** Swift 6, SwiftUI, `@Observable`/`@MainActor`, Swift Testing (`import Testing`), `UserDefaults` persistence, local ShroomKit Swift package.

## Global Constraints

- **Calm/meditative UX** — no arcade or competitive framing, no ranks, no initials.
- **Offline, no accounts, no ads** — persistence stays local in `UserDefaults`.
- **Whisper rule** — the win card *always* shows the completion time in its subtitle (a clock, not competition). On a clear that beats a pre-existing best, it additionally shows a quiet "Your fastest yet" whisper line. First clears and slower clears show no whisper.
- **Persistence reset is acceptable** — the new storage key (`rootline_stats_v1`) is intentionally separate from the old (`rootline_scores_v1`); old arcade data is not migrated (cleared-count can't be reconstructed from a top-5 list, and this is pre-launch dev data). The old key is left orphaned.
- **Minimum tooling:** Xcode 16+ (Swift Testing). All Swift files live in the `rootline` app target except tests, which live in a new `rootlineTests` unit-test target.
- **Build/test command (run from `rootline/`):**
  ```bash
  xcodebuild test -project rootline.xcodeproj -scheme rootline \
    -destination 'platform=iOS Simulator,name=iPhone 16'
  ```
  If that simulator isn't installed, list available ones with `xcrun simctl list devices available` and substitute a name.

---

## File Structure

| File | Action | Responsibility after change |
| --- | --- | --- |
| `rootline/Storage/ScoreStore.swift` | Rewrite | Per-tier `TierStat`; `record`/query API; injectable `UserDefaults`; persistence |
| `rootlineTests/ScoreStoreTests.swift` | Create | Unit tests for `ScoreStore` logic |
| `rootline/Views/WinEntrySheet.swift` | Delete | (gone — no initials entry) |
| `rootline/Views/BestTimesView.swift` | Delete | (replaced by `StatsView`) |
| `rootline/Views/StatsView.swift` | Create | Calm per-tier stats screen |
| `rootline/Views/WinCard.swift` | Modify | Quiet-on-time subtitle + optional "fastest yet" whisper |
| `rootline/Views/PlayView.swift` | Modify | Record silently on solve; drive whisper; remove entry-sheet machinery |
| `rootline/Views/HomeView.swift` | Modify | "Stats" button instead of "Best times" |
| `rootline/Views/RootView.swift` | Modify | `.stats` screen + `openStats()`; wire `StatsView` |

The `extension Int { var asTimerString }` at the bottom of `PlayView.swift` is shared (used by `WinCard` and `StatsView`); it stays put.

---

## Task 1: ScoreStore rewrite (TDD) + silent recording + win-card whisper

Delivers working behavior end-to-end: solving records stats silently, the win card whispers on a new best, the initials sheet is gone, and a plain (un-styled) `StatsView` shows the numbers. The calm visual polish of `StatsView` is Task 2.

**Files:**
- Create: `rootlineTests/ScoreStoreTests.swift`
- Rewrite: `rootline/Storage/ScoreStore.swift`
- Delete: `rootline/Views/WinEntrySheet.swift`
- Delete: `rootline/Views/BestTimesView.swift`
- Create: `rootline/Views/StatsView.swift` (minimal; polished in Task 2)
- Modify: `rootline/Views/WinCard.swift`
- Modify: `rootline/Views/PlayView.swift`
- Modify: `rootline/Views/HomeView.swift`
- Modify: `rootline/Views/RootView.swift`

**Interfaces:**
- Produces (consumed by Task 2 and by views in this task):
  - `struct TierStat: Codable, Equatable, Sendable { var bestSeconds: Int?; var clearedCount: Int }`
  - `enum ClearOutcome: Equatable, Sendable { case firstClear, newBest, noImprovement }`
  - `ScoreStore.init(defaults: UserDefaults = .standard)`
  - `ScoreStore.stat(for: Tier) -> TierStat`
  - `ScoreStore.bestSeconds(for: Tier) -> Int?`
  - `ScoreStore.clearedCount(for: Tier) -> Int`
  - `ScoreStore.hasAnyStats: Bool`
  - `ScoreStore.totalCleared: Int`
  - `@discardableResult ScoreStore.record(seconds: Int, for: Tier) -> ClearOutcome`
  - `ScoreStore.clearAll()`
  - `StatsView(scoreStore: ScoreStore, onClose: () -> Void)`
  - `WinCard(board: Board, fastestYet: Bool, onNext: () -> Void, onMenu: () -> Void)`
  - `AppState.openStats()`, `Screen.stats`

---

- [ ] **Step 1: Create the unit-test target (manual, in Xcode)**

This one-time step must be done in Xcode's UI (the project file is fragile to hand-edit).

1. Open `rootline/rootline.xcodeproj` in Xcode.
2. Menu: **File → New → Target…**
3. Under the **Test** section choose **Unit Testing Bundle**, click **Next**.
4. Set **Product Name:** `rootlineTests`. **Testing System:** *Swift Testing*. **Target to be Tested:** `rootline`. **Language:** Swift. Click **Finish**.
5. Xcode creates a `rootlineTests` group containing a sample `rootlineTests.swift`. Delete that sample file (Move to Trash) so it doesn't clash with the test we add next.
6. Confirm the scheme `rootline` now lists `rootlineTests` under **Test** (Product → Scheme → Edit Scheme → Test → Info).

- [ ] **Step 2: Write the failing tests**

Create `rootline/rootlineTests/ScoreStoreTests.swift`:

```swift
import Testing
import Foundation
@testable import rootline

@MainActor
struct ScoreStoreTests {
    /// Fresh, isolated defaults per store so persistence never leaks between tests.
    private func makeStore() -> ScoreStore {
        let defaults = UserDefaults(suiteName: "test-\(UUID().uuidString)")!
        return ScoreStore(defaults: defaults)
    }

    @Test func firstClearReportsFirstClearAndSetsBest() {
        let store = makeStore()
        let outcome = store.record(seconds: 90, for: .sprout)
        #expect(outcome == .firstClear)
        #expect(store.bestSeconds(for: .sprout) == 90)
        #expect(store.clearedCount(for: .sprout) == 1)
    }

    @Test func fasterClearReportsNewBest() {
        let store = makeStore()
        _ = store.record(seconds: 90, for: .sprout)
        let outcome = store.record(seconds: 60, for: .sprout)
        #expect(outcome == .newBest)
        #expect(store.bestSeconds(for: .sprout) == 60)
        #expect(store.clearedCount(for: .sprout) == 2)
    }

    @Test func slowerClearReportsNoImprovementAndKeepsBest() {
        let store = makeStore()
        _ = store.record(seconds: 60, for: .sprout)
        let outcome = store.record(seconds: 90, for: .sprout)
        #expect(outcome == .noImprovement)
        #expect(store.bestSeconds(for: .sprout) == 60)
        #expect(store.clearedCount(for: .sprout) == 2)
    }

    @Test func equalTimeIsNoImprovement() {
        let store = makeStore()
        _ = store.record(seconds: 60, for: .sprout)
        let outcome = store.record(seconds: 60, for: .sprout)
        #expect(outcome == .noImprovement)
        #expect(store.bestSeconds(for: .sprout) == 60)
    }

    @Test func tiersAreIndependent() {
        let store = makeStore()
        _ = store.record(seconds: 60, for: .sprout)
        #expect(store.bestSeconds(for: .mycelium) == nil)
        #expect(store.clearedCount(for: .mycelium) == 0)
    }

    @Test func statsPersistAcrossInstances() {
        let defaults = UserDefaults(suiteName: "test-\(UUID().uuidString)")!
        let first = ScoreStore(defaults: defaults)
        _ = first.record(seconds: 75, for: .ancient)
        let second = ScoreStore(defaults: defaults)
        #expect(second.bestSeconds(for: .ancient) == 75)
        #expect(second.clearedCount(for: .ancient) == 1)
    }

    @Test func clearAllResetsEverything() {
        let store = makeStore()
        _ = store.record(seconds: 60, for: .sprout)
        _ = store.record(seconds: 80, for: .mycelium)
        store.clearAll()
        #expect(store.bestSeconds(for: .sprout) == nil)
        #expect(store.clearedCount(for: .sprout) == 0)
        #expect(store.hasAnyStats == false)
    }

    @Test func hasAnyStatsReflectsClears() {
        let store = makeStore()
        #expect(store.hasAnyStats == false)
        _ = store.record(seconds: 60, for: .sprout)
        #expect(store.hasAnyStats == true)
    }

    @Test func totalClearedSumsAcrossTiers() {
        let store = makeStore()
        #expect(store.totalCleared == 0)
        _ = store.record(seconds: 60, for: .sprout)
        _ = store.record(seconds: 70, for: .sprout)
        _ = store.record(seconds: 80, for: .mycelium)
        #expect(store.totalCleared == 3)
    }
}
```

- [ ] **Step 3: Run tests to verify they fail to build**

Run (from `rootline/`):
```bash
xcodebuild test -project rootline.xcodeproj -scheme rootline \
  -destination 'platform=iOS Simulator,name=iPhone 16'
```
Expected: BUILD FAILURE — the test references `ScoreStore.init(defaults:)`, `record`, `bestSeconds`, `clearedCount`, `hasAnyStats`, `totalCleared`, `ClearOutcome`, none of which exist yet.

- [ ] **Step 4: Rewrite ScoreStore**

Replace the entire contents of `rootline/Storage/ScoreStore.swift`:

```swift
import Foundation

/// Quiet per-tier stats: the player's fastest clear and how many they've cleared.
struct TierStat: Codable, Equatable, Sendable {
    var bestSeconds: Int? = nil
    var clearedCount: Int = 0
}

/// Result of recording a cleared puzzle, used to decide the win-card whisper.
enum ClearOutcome: Equatable, Sendable {
    case firstClear      // first clear of this tier; no prior best existed
    case newBest         // beat the previous best time
    case noImprovement   // cleared, but not faster than the existing best
}

@MainActor
@Observable
final class ScoreStore {
    private static let key = "rootline_stats_v1"
    private let defaults: UserDefaults

    private(set) var stats: [Tier: TierStat] = Tier.allCases.reduce(into: [:]) { $0[$1] = TierStat() }

    init(defaults: UserDefaults = .standard) {
        self.defaults = defaults
        load()
    }

    func stat(for tier: Tier) -> TierStat {
        stats[tier] ?? TierStat()
    }

    func bestSeconds(for tier: Tier) -> Int? {
        stat(for: tier).bestSeconds
    }

    func clearedCount(for tier: Tier) -> Int {
        stat(for: tier).clearedCount
    }

    /// True when the player has cleared at least one puzzle in any tier.
    var hasAnyStats: Bool {
        stats.values.contains { $0.clearedCount > 0 }
    }

    /// Total puzzles cleared across every tier.
    var totalCleared: Int {
        stats.values.reduce(0) { $0 + $1.clearedCount }
    }

    /// Record a cleared puzzle: bump the completion count, lower the best time
    /// if this was faster, and report what happened so callers can decide
    /// whether to whisper "your fastest yet".
    @discardableResult
    func record(seconds: Int, for tier: Tier) -> ClearOutcome {
        var s = stat(for: tier)
        s.clearedCount += 1
        let outcome: ClearOutcome
        if let best = s.bestSeconds {
            if seconds < best {
                s.bestSeconds = seconds
                outcome = .newBest
            } else {
                outcome = .noImprovement
            }
        } else {
            s.bestSeconds = seconds
            outcome = .firstClear
        }
        stats[tier] = s
        persist()
        return outcome
    }

    func clearAll() {
        for tier in Tier.allCases { stats[tier] = TierStat() }
        persist()
    }

    // MARK: - Persistence

    private struct Stored: Codable {
        var sprout: TierStat = TierStat()
        var mycelium: TierStat = TierStat()
        var ancient: TierStat = TierStat()
        var oldGrowth: TierStat = TierStat()
    }

    private func load() {
        guard let data = defaults.data(forKey: Self.key),
              let stored = try? JSONDecoder().decode(Stored.self, from: data) else { return }
        stats[.sprout]    = stored.sprout
        stats[.mycelium]  = stored.mycelium
        stats[.ancient]   = stored.ancient
        stats[.oldGrowth] = stored.oldGrowth
    }

    private func persist() {
        let stored = Stored(
            sprout:    stat(for: .sprout),
            mycelium:  stat(for: .mycelium),
            ancient:   stat(for: .ancient),
            oldGrowth: stat(for: .oldGrowth)
        )
        if let data = try? JSONEncoder().encode(stored) {
            defaults.set(data, forKey: Self.key)
        }
    }
}
```

(The app target will not fully build yet — consumers still call the old API. The remaining steps fix every call site; run the test command again only after Step 9.)

- [ ] **Step 5: Delete the entry sheet**

```bash
git rm rootline/rootline/Views/WinEntrySheet.swift
```
(If Xcode still lists it, remove the now-dangling file reference: select it in the navigator → Delete → Remove Reference.)

- [ ] **Step 6: Update WinCard — quiet subtitle + whisper**

Replace the contents of `rootline/Views/WinCard.swift`:

```swift
import SwiftUI
import ShroomKit

struct WinCard: View {
    let board: Board
    /// True when this clear beat a pre-existing best time for the tier.
    let fastestYet: Bool
    let onNext: () -> Void
    let onMenu: () -> Void

    @Environment(\.palette) private var palette

    var body: some View {
        VStack(spacing: 14) {
            HStack(spacing: 12) {
                RoundedRectangle(cornerRadius: 12, style: .continuous)
                    .fill(palette.tierSelBg)
                    .frame(width: 44, height: 44)
                    .overlay(
                        Image(systemName: "checkmark")
                            .font(.system(.title3, design: .rounded).weight(.bold))
                            .foregroundStyle(palette.accent)
                    )
                VStack(alignment: .leading, spacing: 2) {
                    Text("Network connected!")
                        .font(.system(.callout, design: .rounded).weight(.semibold))
                        .foregroundStyle(palette.text)
                    Text(subtitle)
                        .font(.system(.footnote, design: .rounded))
                        .foregroundStyle(palette.sub)
                    if fastestYet {
                        Text("Your fastest yet")
                            .font(.system(.caption, design: .rounded).weight(.medium))
                            .foregroundStyle(palette.accent)
                    }
                }
                Spacer(minLength: 0)
            }
            HStack(spacing: 10) {
                Button(action: onMenu) {
                    Text("Menu")
                        .font(.system(.body, design: .rounded).weight(.semibold))
                        .foregroundStyle(palette.text)
                        .frame(maxWidth: .infinity)
                        .frame(minHeight: 44)
                        .padding(.vertical, 8)
                        .background(
                            RoundedRectangle(cornerRadius: 14, style: .continuous)
                                .fill(palette.tierBg)
                                .overlay(
                                    RoundedRectangle(cornerRadius: 14, style: .continuous)
                                        .strokeBorder(palette.tierBorder, lineWidth: 1)
                                )
                        )
                        .contentShape(Rectangle())
                }
                .buttonStyle(.plain)
                Button(action: onNext) {
                    Text("Next puzzle")
                        .font(.system(.body, design: .rounded).weight(.semibold))
                        .foregroundStyle(palette.accentText)
                        .frame(maxWidth: .infinity)
                        .frame(minHeight: 44)
                        .padding(.vertical, 8)
                        .background(
                            RoundedRectangle(cornerRadius: 14, style: .continuous)
                                .fill(palette.accent)
                        )
                        .contentShape(Rectangle())
                }
                .buttonStyle(.plain)
            }
        }
        .padding(.horizontal, 16)
        .padding(.vertical, 14)
        .background(
            RoundedRectangle(cornerRadius: 16, style: .continuous)
                .fill(palette.pill)
        )
    }

    /// Always shows the completion time — a clock, not competition. The
    /// "Your fastest yet" whisper above is the only achievement signal.
    private var subtitle: String {
        let tierLabel = board.tier?.label ?? "Lesson"
        let size = "\(board.puzzle.cols)×\(board.puzzle.rows)"
        let time = board.elapsedSeconds.asTimerString
        return "\(tierLabel) · \(size) · cleared in \(time)"
    }
}
```

- [ ] **Step 7: Update PlayView — record silently, drive the whisper, drop the sheet**

In `rootline/Views/PlayView.swift`:

7a. Replace the four state vars (lines 17–22) — `initials`, `showingWinEntry`, `isNewRecord`, `awaitingEntry` — with a single:

```swift
    @State private var fastestYet: Bool = false
```

7b. Replace the win-card overlay (the `.overlay(alignment: .bottom) { ... }` block) so it no longer checks `awaitingEntry` and passes `fastestYet`:

```swift
        .overlay(alignment: .bottom) {
            if board.isSolved {
                WinCard(board: board, fastestYet: fastestYet, onNext: onNext, onMenu: onMenu)
                    .padding(.horizontal, 18)
                    .padding(.bottom, 18)
                    .transition(.move(edge: .bottom).combined(with: .opacity))
            }
        }
```

7c. Remove the now-unused `awaitingEntry` animation line. Delete:
```swift
        .animation(.spring(response: 0.45, dampingFraction: 0.85), value: awaitingEntry)
```
(Keep the `value: board.isSolved` animation line directly above it.)

7d. Replace the `.onChange(of: board.solveTick)` block with a silent record:

```swift
        .onChange(of: board.solveTick) { _, _ in
            // Guard: swapping boards (Next puzzle) can reset solveTick to 0.
            // Only react to genuine solves.
            guard board.isSolved else { return }
            onClearProgress()
            if let tier = board.tier {
                fastestYet = scoreStore.record(seconds: board.elapsedSeconds, for: tier) == .newBest
            } else {
                fastestYet = false
            }
        }
```

7e. Delete the `.onChange(of: showingWinEntry) { ... }` block entirely.

7f. Delete the entire `.sheet(isPresented: $showingWinEntry) { ... }` block (the WinEntrySheet presentation).

Leave everything else — the `extension Int { var asTimerString }` at the bottom stays.

- [ ] **Step 8: Update HomeView — "Stats" button**

In `rootline/Views/HomeView.swift`:

8a. Rename the closure property (line 8):
```swift
    let onStats: () -> Void
```
8b. Update the button (line 51):
```swift
                    textButton(icon: "chart.bar.fill", title: "Stats", action: onStats)
```

- [ ] **Step 9: Create a minimal StatsView and rewire RootView/AppState**

9a. Delete the old ranked view:
```bash
git rm rootline/rootline/Views/BestTimesView.swift
```
(Remove its Xcode reference too, as in Step 5.)

9b. Create `rootline/Views/StatsView.swift` (minimal — Task 2 adds the calm styling, clear-all, and empty states):

```swift
import SwiftUI
import ShroomKit

struct StatsView: View {
    @Bindable var scoreStore: ScoreStore
    let onClose: () -> Void

    @Environment(\.palette) private var palette

    var body: some View {
        VStack(spacing: 0) {
            HStack(spacing: 12) {
                Button(action: onClose) {
                    Image(systemName: "chevron.left")
                        .font(.system(.body, design: .rounded).weight(.semibold))
                        .foregroundStyle(palette.sub)
                        .frame(minWidth: 44, minHeight: 44)
                        .background(
                            RoundedRectangle(cornerRadius: 12, style: .continuous)
                                .fill(palette.pill)
                        )
                        .contentShape(Rectangle())
                }
                .buttonStyle(.plain)
                Text("Stats")
                    .font(.system(.title2, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.text)
                Spacer()
            }
            .padding(.horizontal, 22)
            .padding(.top, 12)
            .padding(.bottom, 14)

            ScrollView {
                VStack(alignment: .leading, spacing: 14) {
                    ForEach(Tier.allCases) { tier in
                        let stat = scoreStore.stat(for: tier)
                        VStack(alignment: .leading, spacing: 4) {
                            Text(tier.label)
                                .font(.system(.body, design: .rounded).weight(.semibold))
                                .foregroundStyle(palette.text)
                            if let best = stat.bestSeconds {
                                Text("Your fastest: \(best.asTimerString) · \(stat.clearedCount) cleared")
                                    .font(.system(.footnote, design: .rounded))
                                    .foregroundStyle(palette.sub)
                            } else {
                                Text("Not cleared yet")
                                    .font(.system(.footnote, design: .rounded))
                                    .foregroundStyle(palette.sub)
                            }
                        }
                        .frame(maxWidth: .infinity, alignment: .leading)
                    }
                }
                .padding(.horizontal, 22)
                .padding(.bottom, 30)
            }
            .scrollIndicators(.hidden)
        }
        .background(palette.appBg.ignoresSafeArea())
    }
}
```

9c. In `rootline/Views/RootView.swift`, rename the screen case (line 11):
```swift
    case stats
```
9d. Rename the navigation method (lines 102–104):
```swift
    func openStats() {
        screen = .stats
    }
```
9e. Update the HomeView wiring (line 188):
```swift
                onStats: { appState.openStats() },
```
9f. Replace the content switch case (lines 227–232):
```swift
        case .stats:
            StatsView(
                scoreStore: appState.scoreStore,
                onClose: { appState.goHome() }
            )
            .transition(.opacity)
```

- [ ] **Step 10: Run tests and build to verify green**

Run (from `rootline/`):
```bash
xcodebuild test -project rootline.xcodeproj -scheme rootline \
  -destination 'platform=iOS Simulator,name=iPhone 16'
```
Expected: BUILD SUCCEEDED and all 9 `ScoreStoreTests` pass.

- [ ] **Step 11: Manual verification in the simulator**

Launch the app (Xcode Run, or `xcodebuild` + simulator). Confirm:
1. Solve a puzzle in any tier → **no initials entry sheet** appears. The win card slides up directly.
2. Every clear → win card subtitle shows "`Tier · CxR · cleared in M:SS`" (time always present).
3. First clear of a tier → **no** "Your fastest yet" whisper line.
4. Solve the same tier again **faster** → win card adds a "Your fastest yet" line in accent color.
5. Solve again **slower** → no whisper line (just the subtitle time).
6. Home → **Stats** button (chart icon) opens the stats screen; each cleared tier shows fastest time + cleared count; uncleared tiers show "Not cleared yet".

- [ ] **Step 12: Commit**

```bash
cd /Users/michelleweirathmueller/dev/games/shroom-games/rootline
git add -A
git commit -m "feat: replace arcade leaderboard with calm per-tier stats

Rewrite ScoreStore to track best time + cleared count per tier (TDD,
new rootlineTests target). Record silently on solve; win card whispers
only on a new best. Remove WinEntrySheet and ranked BestTimesView;
add StatsView.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Calm visual polish for StatsView

Turn the plain `StatsView` from Task 1 into the calm, finished screen: per-tier cards, the tier meta line, two-line copy, a styled empty state, and a "Clear" affordance with confirmation. No new logic — all data comes from the `ScoreStore` API produced in Task 1.

**Files:**
- Modify: `rootline/Views/StatsView.swift`

**Interfaces:**
- Consumes: `ScoreStore.stat(for:)`, `ScoreStore.hasAnyStats`, `ScoreStore.totalCleared`, `ScoreStore.clearAll()`, `TierStat.bestSeconds`, `TierStat.clearedCount`, `Tier.label`, `Tier.shortMeta`, `Int.asTimerString`.

---

- [ ] **Step 1: Replace StatsView with the polished layout**

Replace the entire contents of `rootline/Views/StatsView.swift`:

```swift
import SwiftUI
import ShroomKit

struct StatsView: View {
    @Bindable var scoreStore: ScoreStore
    let onClose: () -> Void

    @Environment(\.palette) private var palette
    @State private var confirmingClear: Bool = false

    var body: some View {
        VStack(spacing: 0) {
            header
                .padding(.horizontal, 22)
                .padding(.top, 12)
                .padding(.bottom, 14)
            ScrollView {
                VStack(alignment: .leading, spacing: 12) {
                    ForEach(Tier.allCases) { tier in
                        tierCard(tier)
                    }
                    if scoreStore.hasAnyStats {
                        Text("\(scoreStore.totalCleared) puzzles cleared")
                            .font(.system(.footnote, design: .rounded).weight(.medium))
                            .foregroundStyle(palette.sub)
                            .frame(maxWidth: .infinity, alignment: .center)
                            .padding(.top, 6)
                    }
                }
                .padding(.horizontal, 22)
                .padding(.bottom, 30)
            }
            .scrollIndicators(.hidden)
        }
        .background(palette.appBg.ignoresSafeArea())
        .alert("Clear all stats?", isPresented: $confirmingClear) {
            Button("Cancel", role: .cancel) { }
            Button("Clear", role: .destructive) { scoreStore.clearAll() }
        } message: {
            Text("This wipes every fastest time and cleared count across all tiers. Can't be undone.")
        }
    }

    private var header: some View {
        HStack(spacing: 12) {
            Button(action: onClose) {
                Image(systemName: "chevron.left")
                    .font(.system(.body, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.sub)
                    .frame(minWidth: 44, minHeight: 44)
                    .background(
                        RoundedRectangle(cornerRadius: 12, style: .continuous)
                            .fill(palette.pill)
                    )
                    .contentShape(Rectangle())
            }
            .buttonStyle(.plain)
            Text("Stats")
                .font(.system(.title2, design: .rounded).weight(.semibold))
                .foregroundStyle(palette.text)
            Spacer()
            if scoreStore.hasAnyStats {
                Button { confirmingClear = true } label: {
                    Text("Clear")
                        .font(.system(.footnote, design: .rounded).weight(.semibold))
                        .foregroundStyle(palette.sub)
                        .padding(.horizontal, 12)
                        .frame(minHeight: 44)
                        .contentShape(Rectangle())
                }
                .buttonStyle(.plain)
            }
        }
    }

    private func tierCard(_ tier: Tier) -> some View {
        let stat = scoreStore.stat(for: tier)
        return VStack(alignment: .leading, spacing: 6) {
            HStack(alignment: .firstTextBaseline) {
                Text(tier.label.uppercased())
                    .font(.system(.caption2, design: .rounded).weight(.semibold))
                    .tracking(1.3)
                    .foregroundStyle(palette.sub)
                Spacer()
                Text(tier.shortMeta)
                    .font(.system(.caption2, design: .rounded))
                    .foregroundStyle(palette.sub)
            }
            if let best = stat.bestSeconds {
                Text("Your fastest: \(best.asTimerString)")
                    .font(.system(.body, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.text)
                    .monospacedDigit()
                Text("\(stat.clearedCount) cleared")
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
        .background(
            RoundedRectangle(cornerRadius: 14, style: .continuous)
                .fill(palette.pill)
        )
    }
}
```

- [ ] **Step 2: Build to verify it compiles**

Run (from `rootline/`):
```bash
xcodebuild build -project rootline.xcodeproj -scheme rootline \
  -destination 'platform=iOS Simulator,name=iPhone 16'
```
Expected: BUILD SUCCEEDED.

- [ ] **Step 3: Manual verification in the simulator**

1. Open **Stats** from Home. Each tier is a rounded card: UPPERCASE tier label + `CxR` meta on the top row.
2. A cleared tier shows "Your fastest: M:SS" and "N cleared". An uncleared tier shows "Not cleared yet".
3. Below the cards, a centered "**X puzzles cleared**" total appears once at least one tier has stats.
4. The **Clear** button appears in the header only when at least one tier has stats; tapping it shows the confirmation alert; confirming resets every card to "Not cleared yet", removes the total, and hides the Clear button.

- [ ] **Step 4: Commit**

```bash
cd /Users/michelleweirathmueller/dev/games/shroom-games/rootline
git add -A
git commit -m "feat: calm visual polish for StatsView

Per-tier cards with tier meta, two-line fastest/cleared copy, empty
states, and a confirmed clear-all.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Notes for the implementer

- **Why `record` returns an enum, not a `Bool`:** `firstClear` and `noImprovement` both stay quiet on time, but distinguishing them keeps the whisper rule explicit and leaves room for a future first-clear acknowledgment without reworking the API. Only `.newBest` whispers.
- **Recording fires once per solve:** `board.solveTick` only triggers the record while `board.isSolved` is true, and solved boards are swapped out (Next/Menu) rather than re-entered (`finishLaunch` restores only `!isSolved` boards), so a clear is never double-counted.
- **Stale `fastestYet`:** it's only read while `board.isSolved`, and it's recomputed on every solve before the win card shows, so no explicit reset is needed when boards swap.
- **Persistence reset:** see Global Constraints — old `rootline_scores_v1` data is intentionally not migrated.
