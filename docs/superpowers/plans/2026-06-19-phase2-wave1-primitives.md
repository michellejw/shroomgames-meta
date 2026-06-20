# Phase 2 Wave 1 — Shared UI Primitives Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract four duplicated UI primitives (eyebrow label, pill icon button, stat pill, button styles) into ShroomKit with accessibility fixed and tokens adopted, then swap rootline + shroomsweeper onto them.

**Architecture:** Each primitive is a small focused file under `shroomkit/Sources/ShroomKit/Components/`. Buttons are SwiftUI `ButtonStyle`s; the rest are `View`s. Each reads `@Environment(\.palette)` and uses ShroomKit's `Radius`/`FontTracking` tokens. After building a component (verified by `swift build`), both apps' duplicated call sites are swapped onto it and compile-checked with `xcodebuild`. Visual confirmation is a per-component review pause (Michelle, in Xcode).

**Tech Stack:** Swift 6, SwiftUI, ShroomKit (local Swift package), Swift Testing. Three git repos: `shroomkit`, `rootline`, `shroomsweeper`.

## Global Constraints

- iOS 17+; SwiftUI only; 4-space indentation; PascalCase types / camelCase members.
- Views never hardcode colors — always `@Environment(\.palette)`. Tap targets ≥ 44pt.
- Inside extracted components: use `Radius.xl` (16, buttons), `Radius.md` (12, pills/icon buttons), `FontTracking.eyebrow` (1.3). Do NOT sweep literals elsewhere.
- Accessibility baked in: `PillIconButton.accessibilityLabel` required; `EyebrowLabel` uses `.caption` (not `.caption2`) + `.textCase(.uppercase)` on sentence-case input; buttons guarantee `minHeight: 44`.
- No back-compat shims — update ShroomKit + both apps together; delete the replaced local helpers.
- Public API gets one-line doc comments only where the WHY is non-obvious.
- `swift build` (run in `shroomkit/`) is the hard compile gate for the package. App compile checks: `xcodebuild build -scheme <app> -destination 'platform=iOS Simulator,name=iPhone 16'` (run in the app's repo; adjust the simulator name via `xcrun simctl list devices available` if needed).
- Repo paths: `/Users/michelleweirathmueller/dev/games/shroom-games/{shroomkit,rootline,shroomsweeper}`. The suite root is not a git repo — commit inside each repo.

---

## File Structure

| File | Action | Responsibility |
| --- | --- | --- |
| `shroomkit/.../Components/EyebrowLabel.swift` | Create | Tracked-caps section label |
| `shroomkit/.../Components/PillIconButton.swift` | Create | 44pt icon button + `ThemeMode.iconName` |
| `shroomkit/.../Components/StatPill.swift` | Create | Icon + monospaced readout |
| `shroomkit/.../Components/ShroomButtonStyle.swift` | Create | primary/secondary/outline `ButtonStyle` |
| `shroomkit/Tests/ShroomKitTests/ThemeModeIconTests.swift` | Create | Unit-test the `ThemeMode.iconName` helper |
| rootline + shroomsweeper view files | Modify | Swap local implementations onto the new components |

Each task is one component end-to-end (ShroomKit + both app swaps). Independent of the others.

---

## Task 1: EyebrowLabel

**Files:**
- Create: `shroomkit/Sources/ShroomKit/Components/EyebrowLabel.swift`
- Modify (rootline): `HomeView.swift`, `PlayView.swift`, `StatsView.swift`, `SettingsSheet.swift`
- Modify (shroomsweeper): `HomeView.swift`, `TutorialView.swift`

**Interfaces:**
- Produces: `EyebrowLabel(_ text: String, tint: EyebrowLabel.Tint = .sub)`; `enum Tint { case sub, accent }`.

- [ ] **Step 1: Create the component**

`shroomkit/Sources/ShroomKit/Components/EyebrowLabel.swift`:
```swift
import SwiftUI

/// Tracked, uppercased caption used as a quiet section/card header.
/// Pass sentence-case text — uppercasing is visual only, so VoiceOver reads the word.
public struct EyebrowLabel: View {
    public enum Tint { case sub, accent }

    private let text: String
    private let tint: Tint

    @Environment(\.palette) private var palette

    public init(_ text: String, tint: Tint = .sub) {
        self.text = text
        self.tint = tint
    }

    public var body: some View {
        Text(text)
            .font(.system(.caption, design: .rounded).weight(.semibold))
            .tracking(FontTracking.eyebrow)
            .textCase(.uppercase)
            .foregroundStyle(tint == .accent ? palette.accent : palette.sub)
    }
}
```

- [ ] **Step 2: Build ShroomKit**

Run (in `shroomkit/`): `swift build`
Expected: `Build complete!`

- [ ] **Step 3: Swap rootline call sites**

In each location, replace the inline `Text(...).font(.system(.caption2…)).tracking(1.3).foregroundStyle(palette.sub|accent)` eyebrow with `EyebrowLabel(...)`, passing **sentence-case** text (drop the manual `.uppercased()`):
- `HomeView.swift` (~line 67): `Text("DIFFICULTY")…` → `EyebrowLabel("Difficulty")`
- `PlayView.swift` (~line 133): the tier label `Text((board.tier?.label ?? "Lesson").uppercased())…` → `EyebrowLabel(board.tier?.label ?? "Lesson")`
- `StatsView.swift` (~line 80): tier label `Text(tier.label.uppercased())…` → `EyebrowLabel(tier.label)`
- `SettingsSheet.swift` `section(title:)` helper (~line 145): replace the inner `Text(title.uppercased())…` with `EyebrowLabel(title)`.

Delete any now-unused local eyebrow styling. `import ShroomKit` is already present in these files.

- [ ] **Step 4: Swap shroomsweeper call sites**

- `HomeView.swift` (~line 49): `Text("DIFFICULTY")…` → `EyebrowLabel("Difficulty")`
- `TutorialView.swift` (~line 72): step label `Text(flow.stepLabel)…foregroundStyle(palette.accent)` → `EyebrowLabel(flow.stepLabel, tint: .accent)`

- [ ] **Step 5: Compile-check both apps**

Run in `rootline/`: `xcodebuild build -scheme rootline -destination 'platform=iOS Simulator,name=iPhone 16' | tail -3`
Run in `shroomsweeper/`: `xcodebuild build -scheme shroomsweeper -destination 'platform=iOS Simulator,name=iPhone 16' | tail -3`
Expected: `BUILD SUCCEEDED` for both.

- [ ] **Step 6: Commit (3 repos)**

```bash
cd /Users/michelleweirathmueller/dev/games/shroom-games/shroomkit && git add Sources/ShroomKit/Components/EyebrowLabel.swift && git commit -m "feat: add EyebrowLabel component"
cd /Users/michelleweirathmueller/dev/games/shroom-games/rootline && git add -A && git commit -m "refactor: use ShroomKit EyebrowLabel"
cd /Users/michelleweirathmueller/dev/games/shroom-games/shroomsweeper && git add -A && git commit -m "refactor: use ShroomKit EyebrowLabel"
```
(End each commit message with the `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>` trailer.)

- [ ] **Step 7: Review pause (Michelle, Xcode)** — run both apps; confirm eyebrow labels look right (slightly larger now, `.caption`) and nothing else shifted.

---

## Task 2: PillIconButton (+ ThemeMode.iconName)

**Files:**
- Create: `shroomkit/Sources/ShroomKit/Components/PillIconButton.swift`
- Create: `shroomkit/Tests/ShroomKitTests/ThemeModeIconTests.swift`
- Modify (rootline): `PlayView.swift`, `HomeView.swift`, `DifficultyView.swift`, `StatsView.swift`
- Modify (shroomsweeper): `HomeView.swift`, `WelcomeView.swift`, `GameView.swift`

**Interfaces:**
- Consumes: `palette`, `Radius.md`.
- Produces: `PillIconButton(systemName:accessibilityLabel:shape:isEnabled:action:)`, `enum PillIconButton.Shape { case roundedRect, circle }`, and `ThemeMode.iconName: String`.

- [ ] **Step 1: Write the failing test for the icon helper**

`shroomkit/Tests/ShroomKitTests/ThemeModeIconTests.swift`:
```swift
import Testing
@testable import ShroomKit

struct ThemeModeIconTests {
    @Test func iconNamesAreStable() {
        #expect(ThemeMode.system.iconName == "circle.lefthalf.filled")
        #expect(ThemeMode.forest.iconName == "sun.max")
        #expect(ThemeMode.twilight.iconName == "moon.stars")
    }
}
```

- [ ] **Step 2: Run it to verify failure**

Run (in `shroomkit/`): `swift test --filter ThemeModeIconTests`
Expected: FAIL — `ThemeMode` has no `iconName`.

- [ ] **Step 3: Create the component + helper**

`shroomkit/Sources/ShroomKit/Components/PillIconButton.swift`:
```swift
import SwiftUI

public extension ThemeMode {
    /// SF Symbol representing this theme in a cycle control.
    var iconName: String {
        switch self {
        case .system:   return "circle.lefthalf.filled"
        case .forest:   return "sun.max"
        case .twilight: return "moon.stars"
        }
    }
}

/// 44pt icon-only button in a pill (rounded-rect or circle). The
/// `accessibilityLabel` is required so icon buttons are never unlabeled.
public struct PillIconButton: View {
    public enum Shape { case roundedRect, circle }

    private let systemName: String
    private let label: String
    private let shape: Shape
    private let isEnabled: Bool
    private let action: () -> Void

    @Environment(\.palette) private var palette

    public init(systemName: String, accessibilityLabel: String,
                shape: Shape = .roundedRect, isEnabled: Bool = true,
                action: @escaping () -> Void) {
        self.systemName = systemName
        self.label = accessibilityLabel
        self.shape = shape
        self.isEnabled = isEnabled
        self.action = action
    }

    public var body: some View {
        Button(action: action) {
            Image(systemName: systemName)
                .font(.system(.body, design: .rounded).weight(.semibold))
                .foregroundStyle(palette.sub.opacity(isEnabled ? 1 : 0.35))
                .frame(minWidth: 44, minHeight: 44)
                .background(background)
                .contentShape(Rectangle())
        }
        .buttonStyle(.plain)
        .disabled(!isEnabled)
        .accessibilityLabel(label)
    }

    @ViewBuilder private var background: some View {
        switch shape {
        case .roundedRect:
            RoundedRectangle(cornerRadius: Radius.md, style: .continuous).fill(palette.pill)
        case .circle:
            Circle().fill(palette.pill)
        }
    }
}
```
(`.contentShape(Rectangle())` is enough — the 44×44 frame is the hit area regardless of the visual corner rounding.)

- [ ] **Step 4: Run the test + build**

Run (in `shroomkit/`): `swift test --filter ThemeModeIconTests && swift build`
Expected: test PASS, `Build complete!`

- [ ] **Step 5: Swap rootline call sites**

Replace each toolbar icon button (the `Image(systemName:).font(...).foregroundStyle(palette.sub).frame(minWidth:44,minHeight:44).background(RoundedRectangle(cornerRadius:12|14…).fill(palette.pill)).contentShape(...)` wrapped in a plain `Button`) with `PillIconButton`:
- `PlayView.swift`: back (`chevron.left`, label "Back"), theme cycle (`themeIconName(for:settings.themeMode)` → `settings.themeMode.iconName`, label "Theme"), hint (`questionmark`, label "Hint", `isEnabled: board.allowHints && board.hintsRemaining > 0 && !board.isSolved`), reveal/eye (label "Show solution", with its existing enabled condition).
- `HomeView.swift`: settings gear (`gearshape`, label "Settings", `shape: .roundedRect`).
- `DifficultyView.swift`, `StatsView.swift`: back (`chevron.left`, label "Back").

Example (rootline `DifficultyView` back button) becomes:
```swift
PillIconButton(systemName: "chevron.left", accessibilityLabel: "Back", action: onBack)
```
Delete the local `themeIconName(for:)` free functions in both apps (use `ThemeMode.iconName`). Keep the surrounding layout (spacers, etc.) intact.

- [ ] **Step 6: Swap shroomsweeper call sites**

- `HomeView.swift` + `WelcomeView.swift`: the theme-cycle circle button → `PillIconButton(systemName: themeMode.iconName, accessibilityLabel: "Theme", shape: .circle, action: onCycleTheme)`. Delete the shared `themeIconName(for:)` free function (now `ThemeMode.iconName`).
- `GameView.swift` title-row icon buttons → `PillIconButton(...)` with appropriate `systemName`/label.

- [ ] **Step 7: Compile-check both apps**

Same `xcodebuild build` commands as Task 1 Step 5. Expected: `BUILD SUCCEEDED` both.

- [ ] **Step 8: Commit (3 repos)** — as Task 1 Step 6, messages: ShroomKit "feat: add PillIconButton + ThemeMode.iconName"; apps "refactor: use ShroomKit PillIconButton".

- [ ] **Step 9: Review pause (Michelle)** — run both apps; tap each icon button; confirm look + that disabled hint/reveal still dim correctly; VoiceOver now announces labels.

---

## Task 3: StatPill

**Files:**
- Create: `shroomkit/Sources/ShroomKit/Components/StatPill.swift`
- Modify (rootline): `PlayView.swift`
- Modify (shroomsweeper): `GameView.swift`

**Interfaces:**
- Produces: `StatPill(_ text: String, systemName: String? = nil)`.

- [ ] **Step 1: Create the component**

`shroomkit/Sources/ShroomKit/Components/StatPill.swift`:
```swift
import SwiftUI

/// Non-interactive HUD readout: optional SF Symbol + monospaced-digit text in a pill.
public struct StatPill: View {
    private let text: String
    private let systemName: String?

    @Environment(\.palette) private var palette

    public init(_ text: String, systemName: String? = nil) {
        self.text = text
        self.systemName = systemName
    }

    public var body: some View {
        HStack(spacing: 6) {
            if let systemName {
                Image(systemName: systemName)
                    .font(.system(.caption, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.sub)
            }
            Text(text)
                .font(.system(.subheadline, design: .rounded).weight(.semibold))
                .foregroundStyle(palette.text)
                .monospacedDigit()
        }
        .padding(.horizontal, 12)
        .padding(.vertical, 7)
        .background(
            RoundedRectangle(cornerRadius: Radius.md, style: .continuous)
                .fill(palette.pill)
        )
    }
}
```

- [ ] **Step 2: Build ShroomKit** — `swift build` → `Build complete!`

- [ ] **Step 3: Swap rootline**

`PlayView.swift`: replace the `statPill(systemName:text:)` helper usage (the timer pill) with `StatPill(board.elapsedSeconds.asTimerString, systemName: "clock")`. Delete the local `statPill` helper. Leave `hintsPill` (dot indicator) as-is — out of scope this wave.

- [ ] **Step 4: Swap shroomsweeper**

`GameView.swift`: replace the `statPill(icon:text:)` usages with `StatPill(text, systemName: ...)` where the leading icon is an SF Symbol. If a usage passes a non-symbol custom icon view, leave that one local (note it) — the convenience init is symbol-only by design.

- [ ] **Step 5: Compile-check both apps** — `xcodebuild build` both → `BUILD SUCCEEDED`.

- [ ] **Step 6: Commit (3 repos)** — ShroomKit "feat: add StatPill"; apps "refactor: use ShroomKit StatPill".

- [ ] **Step 7: Review pause (Michelle)** — confirm the play/game HUD pills look unchanged.

---

## Task 4: Button styles (primary / secondary / outline)

**Files:**
- Create: `shroomkit/Sources/ShroomKit/Components/ShroomButtonStyle.swift`
- Modify (shroomkit): `Components/WelcomeScaffold.swift` (swap its inline buttons onto the styles)
- Modify (rootline): `HomeView.swift`, `TutorialView.swift`, `WinCard.swift`, `RevealedCard.swift`
- Modify (shroomsweeper): `HomeView.swift`, `OptionsSheet.swift`, `TutorialView.swift`, `ResultBar.swift`, `WinEntryCard.swift`

**Interfaces:**
- Produces: `.buttonStyle(.shroomPrimary)`, `.shroomPrimary(prominent:)`, `.shroomSecondary`, `.shroomOutline`.

- [ ] **Step 1: Create the button style**

`shroomkit/Sources/ShroomKit/Components/ShroomButtonStyle.swift`:
```swift
import SwiftUI

/// Full-width CTA styles. Apply to a `Button` whose label is a plain `Text`.
public struct ShroomButtonStyle: ButtonStyle {
    public enum Kind { case primary, secondary, outline }

    let kind: Kind
    let prominent: Bool

    @Environment(\.palette) private var palette

    public func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .font(.system(prominent ? .title3 : .headline, design: .rounded).weight(.semibold))
            .foregroundStyle(foreground)
            .frame(maxWidth: .infinity)
            .frame(minHeight: 44)
            .padding(.vertical, prominent ? 14 : 10)
            .background(background)
            .contentShape(Rectangle())
            .opacity(configuration.isPressed ? 0.9 : 1)
            .scaleEffect(configuration.isPressed ? 0.98 : 1)
            .animation(.easeOut(duration: 0.12), value: configuration.isPressed)
    }

    private var foreground: Color {
        switch kind {
        case .primary:  return palette.accentText
        case .secondary, .outline: return palette.text
        }
    }

    @ViewBuilder private var background: some View {
        let shape = RoundedRectangle(cornerRadius: Radius.xl, style: .continuous)
        switch kind {
        case .primary:   shape.fill(palette.accent)
        case .secondary: shape.fill(palette.pill)
        case .outline:   shape.fill(palette.tierBg).overlay(shape.strokeBorder(palette.tierBorder, lineWidth: 1))
        }
    }
}

public extension ButtonStyle where Self == ShroomButtonStyle {
    static var shroomPrimary: ShroomButtonStyle { .init(kind: .primary, prominent: false) }
    static func shroomPrimary(prominent: Bool) -> ShroomButtonStyle { .init(kind: .primary, prominent: prominent) }
    static var shroomSecondary: ShroomButtonStyle { .init(kind: .secondary, prominent: false) }
    static var shroomOutline: ShroomButtonStyle { .init(kind: .outline, prominent: false) }
}
```

- [ ] **Step 2: Build ShroomKit** — `swift build` → `Build complete!`

- [ ] **Step 3: Swap WelcomeScaffold (in ShroomKit)**

In `WelcomeScaffold.swift`, replace the two inline `Button { } label: { Text(...).…background(RoundedRectangle…) }.buttonStyle(.plain)` blocks with:
```swift
Button(primaryLabel, action: onPrimary).buttonStyle(.shroomPrimary)
Button(secondaryLabel, action: onSecondary).buttonStyle(.shroomSecondary)
```
Re-run `swift build` → `Build complete!`.

- [ ] **Step 4: Swap rootline call sites**

Replace each full-width CTA's inline styling with the style modifier; delete local button helpers:
- `HomeView.swift`: the `primaryButton(_:action:)` helper → call site becomes `Button("Play", action: onPlay).buttonStyle(.shroomPrimary(prominent: true))`. Delete `primaryButton`.
- `TutorialView.swift`: the unlock-strip CTA → `.buttonStyle(.shroomPrimary)`.
- `WinCard.swift` + `RevealedCard.swift`: the "Menu" button → `Button("Menu", action: onMenu).buttonStyle(.shroomOutline)`; the primary ("Next puzzle"/equivalent) → `.buttonStyle(.shroomPrimary)`. Delete the inline button chrome.

- [ ] **Step 5: Swap shroomsweeper call sites**

- `HomeView.swift` ("Play"), `OptionsSheet.swift` ("Start new game"), `TutorialView.swift` ("Got it"/"Start foraging"), `WinEntryCard.swift` ("Save") → `.buttonStyle(.shroomPrimary)` (use `prominent: true` only for the big Home "Play").
- `ResultBar.swift`: secondary "Menu" → `.shroomOutline`; primary "Play again"/"Try again" → `.shroomPrimary`.
Delete the replaced inline button chrome in each.

- [ ] **Step 6: Compile-check both apps** — `xcodebuild build` both → `BUILD SUCCEEDED`.

- [ ] **Step 7: Commit (3 repos)** — ShroomKit "feat: add ShroomButtonStyle (primary/secondary/outline); adopt in WelcomeScaffold"; apps "refactor: use ShroomKit button styles".

- [ ] **Step 8: Review pause (Michelle)** — run both apps; confirm CTAs look right with the unified `Radius.xl`, the new pressed-state feels good, and the big Home "Play" still reads prominent.

---

## Notes for the implementer

- **`@Environment` in `ButtonStyle`:** valid in SwiftUI — `makeBody` is a view context, so `@Environment(\.palette)` resolves. (WelcomeScaffold already proves palette is in the environment.)
- **`contentShape(Rectangle())`** is sufficient for the icon button hit area (the 44×44 frame is the target); don't over-engineer a shape-matched hit region.
- **Sentence-case eyebrow:** the visual uppercasing is `.textCase(.uppercase)`; pass `"Difficulty"`, never `"DIFFICULTY"`.
- **Deleting local helpers:** after a swap, remove the now-dead private helper funcs (`primaryButton`, `statPill`, `themeIconName`, inline eyebrow) so the call site is the only definition — no shims.
- **If a swap feels awkward** (the ShroomKit validation rule), stop and report it — the API needs adjusting before continuing, not a workaround at the call site.
- **xcodebuild is slow** (~1–2 min/app); run it once per task after both apps are swapped, not per edit.
