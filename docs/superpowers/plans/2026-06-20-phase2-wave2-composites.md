# Phase 2 Wave 2 — Composite Components Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract five composite UI components (ScreenHeader, SegmentedToggle, SelectionCard, ResultCard, settings chrome) into ShroomKit composing the Wave 1 primitives, with selection-state accessibility added, then swap rootline + shroomsweeper onto them.

**Architecture:** Each composite is a focused file under `shroomkit/Sources/ShroomKit/Components/`, reading `@Environment(\.palette)` and using `Radius`/`FontTracking` tokens. ScreenHeader/SegmentedToggle/SelectionCard/SettingsChrome compose `PillIconButton`/`EyebrowLabel`; ResultCard composes `ShroomButtonStyle`. After each component (verified by `swift build`), both apps' call sites are swapped and compile-checked with `xcodebuild`. Visual confirmation is a per-component review pause (Michelle).

**Tech Stack:** Swift 6, SwiftUI, ShroomKit. Three repos: `shroomkit`, `rootline`, `shroomsweeper`.

## Global Constraints

- iOS 17+; SwiftUI only; 4-space indent; PascalCase types / camelCase members. Views read `@Environment(\.palette)`; tap targets ≥ 44pt.
- Use tokens **inside the components only**: `Radius.xl` (cards/outer toggle/buttons), `Radius.md` (inner toggle segment, chips), `Radius.lg` (settings row), `FontTracking.eyebrow` (via `EyebrowLabel`). No literal sweep elsewhere.
- Accessibility baked in: `.accessibilityAddTraits(.isSelected)` on the active `SegmentedToggle` segment, selected `SelectionCard`, active `SelectionChip`.
- No back-compat shims; update kit + both apps together; delete replaced local helpers.
- ShroomKit hard gate: `swift build` (run in `shroomkit/`). App compile: `xcodebuild build -project <app>.xcodeproj -scheme <app> -destination 'generic/platform=iOS Simulator'` (run in the app repo; grep for `BUILD SUCCEEDED`).
- **shroomsweeper commits: stage only the edited Swift files** (`git add <files>`), never `git add -A` — the repo may carry unrelated working-tree noise (xcuserdata).
- Repo paths: `/Users/michelleweirathmueller/dev/games/shroom-games/{shroomkit,rootline,shroomsweeper}`. Commit inside each repo; end messages with the `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>` trailer.
- Each task ends with a **review pause** (Michelle, Xcode) before the next.

---

## File Structure

| File | Action | Responsibility |
| --- | --- | --- |
| `Components/ScreenHeader.swift` | Create | Back + title (+ optional subtitle) + trailing slot |
| `Components/SelectionCard.swift` | Create | Radio selection row |
| `Components/SegmentedToggle.swift` | Create | Generic N-segment pill toggle |
| `Components/ResultCard.swift` | Create | Badge + title/subtitle/note + button pair |
| `Components/SettingsChrome.swift` | Create | `SettingsSection` + `SelectionChip` + `SettingsRow` |
| app view files | Modify | Swap local implementations onto the components |

Each task is one component end-to-end (ShroomKit + both app swaps), independent of the others.

---

## Task 1: ScreenHeader

**Files:**
- Create: `shroomkit/Sources/ShroomKit/Components/ScreenHeader.swift`
- Modify (rootline): `DifficultyView.swift`, `StatsView.swift`
- Modify (shroomsweeper): any simple back+title header (e.g. an options/scores back header if present; otherwise none — note it)

**Interfaces:**
- Consumes: `PillIconButton`, `EyebrowLabel`, `palette`.
- Produces: `ScreenHeader(_ title: String, subtitle: String? = nil, onBack: (() -> Void)?, @ViewBuilder trailing: () -> some View = { EmptyView() })`.

- [ ] **Step 1: Create the component**

`shroomkit/Sources/ShroomKit/Components/ScreenHeader.swift`:
```swift
import SwiftUI

/// Screen header skeleton: optional back button + title (+ optional eyebrow
/// subtitle) + a trailing slot for screen-specific actions.
public struct ScreenHeader<Trailing: View>: View {
    private let title: String
    private let subtitle: String?
    private let onBack: (() -> Void)?
    private let trailing: Trailing

    @Environment(\.palette) private var palette

    public init(_ title: String, subtitle: String? = nil, onBack: (() -> Void)?,
                @ViewBuilder trailing: () -> Trailing = { EmptyView() }) {
        self.title = title
        self.subtitle = subtitle
        self.onBack = onBack
        self.trailing = trailing()
    }

    public var body: some View {
        HStack(spacing: 12) {
            if let onBack {
                PillIconButton(systemName: "chevron.left", accessibilityLabel: "Back", action: onBack)
            }
            VStack(alignment: .leading, spacing: 1) {
                if let subtitle { EyebrowLabel(subtitle) }
                Text(title)
                    .font(.system(.title2, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.text)
            }
            Spacer()
            trailing
        }
    }
}
```

- [ ] **Step 2: Build ShroomKit** — `swift build` → `Build complete!`

- [ ] **Step 3: Swap rootline**
- `DifficultyView.swift`: replace the `header` HStack (back button + "Difficulty" title + Spacer) with `ScreenHeader("Difficulty", onBack: onBack)`. Delete the now-dead local header code.
- `StatsView.swift`: replace the `header` (back + "Stats" + Spacer + conditional "Clear") with:
  ```swift
  ScreenHeader("Stats", onBack: onClose) {
      if scoreStore.hasAnyStats {
          Button("Clear") { confirmingClear = true }
              .font(.system(.footnote, design: .rounded).weight(.semibold))
              .foregroundStyle(palette.sub)
              .frame(minHeight: 44)
      }
  }
  ```
  Keep the surrounding padding the call site applies to the header.

- [ ] **Step 4: Swap shroomsweeper** — search for a simple back+title header (`OptionsSheet`/scores). If one exists, swap it; if shroomsweeper has no plain back+title header (its game header is bespoke and deferred), note "no shroomsweeper adoption for ScreenHeader this task" in the report and proceed.

- [ ] **Step 5: Compile-check** — `xcodebuild build` rootline (+ shroomsweeper if changed) → `BUILD SUCCEEDED`.

- [ ] **Step 6: Commit** (shroomkit + rootline [+ shroomsweeper if changed]) — ShroomKit "feat: add ScreenHeader"; apps "refactor: use ShroomKit ScreenHeader".

- [ ] **Step 7: Review pause (Michelle)** — Difficulty + Stats headers unchanged; Stats "Clear" still works.

---

## Task 2: SelectionCard

**Files:**
- Create: `shroomkit/Sources/ShroomKit/Components/SelectionCard.swift`
- Modify (rootline): `DifficultyView.swift`
- Modify (shroomsweeper): `OptionsSheet.swift`

**Interfaces:**
- Produces: `SelectionCard(title: String, subtitle: String, isSelected: Bool, action: @escaping () -> Void)`.

- [ ] **Step 1: Create the component**

`shroomkit/Sources/ShroomKit/Components/SelectionCard.swift`:
```swift
import SwiftUI

/// Tappable radio-style selection row: title + subtitle + radio dot, with
/// selected chrome and the `.isSelected` accessibility trait.
public struct SelectionCard: View {
    private let title: String
    private let subtitle: String
    private let isSelected: Bool
    private let action: () -> Void

    @Environment(\.palette) private var palette

    public init(title: String, subtitle: String, isSelected: Bool, action: @escaping () -> Void) {
        self.title = title
        self.subtitle = subtitle
        self.isSelected = isSelected
        self.action = action
    }

    public var body: some View {
        Button(action: action) {
            HStack(spacing: 12) {
                VStack(alignment: .leading, spacing: 2) {
                    Text(title)
                        .font(.system(.headline, design: .rounded))
                        .foregroundStyle(palette.text)
                    Text(subtitle)
                        .font(.system(.subheadline, design: .rounded))
                        .foregroundStyle(palette.sub)
                }
                Spacer()
                ZStack {
                    Circle()
                        .stroke(isSelected ? palette.accent : palette.tierBorder, lineWidth: 2)
                        .frame(width: 22, height: 22)
                    if isSelected {
                        Circle().fill(palette.accent).frame(width: 12, height: 12)
                    }
                }
            }
            .padding(.horizontal, 18)
            .padding(.vertical, 15)
            .frame(minHeight: 44)
            .background(
                RoundedRectangle(cornerRadius: Radius.xl, style: .continuous)
                    .fill(isSelected ? palette.tierSelBg : palette.tierBg)
                    .overlay(
                        RoundedRectangle(cornerRadius: Radius.xl, style: .continuous)
                            .strokeBorder(isSelected ? palette.accent : palette.tierBorder, lineWidth: 2)
                    )
            )
            .contentShape(Rectangle())
        }
        .buttonStyle(.plain)
        .accessibilityAddTraits(isSelected ? .isSelected : [])
    }
}
```

- [ ] **Step 2: Build ShroomKit** — `swift build`.

- [ ] **Step 3: Swap rootline** — `DifficultyView.swift`: replace the `tierRow(_:)` helper's body with `SelectionCard(title: tier.label, subtitle: tier.meta, isSelected: tier == selected) { onPick(tier) }` (adjust property/closure names to the file). Delete the local row chrome.

- [ ] **Step 4: Swap shroomsweeper** — `OptionsSheet.swift`: replace `difficultyRow(_:)` with `SelectionCard(title: difficulty.label, subtitle: difficulty.sizeDescription, isSelected: ...) { ... }`. Delete the local chrome.

- [ ] **Step 5: Compile-check** both apps → `BUILD SUCCEEDED`.

- [ ] **Step 6: Commit** (3 repos).

- [ ] **Step 7: Review pause (Michelle)** — Difficulty picker (both apps) unchanged; selected state reads correctly; VoiceOver announces "selected".

---

## Task 3: SegmentedToggle

**Files:**
- Create: `shroomkit/Sources/ShroomKit/Components/SegmentedToggle.swift`
- Modify (rootline): `PlayView.swift`, `TutorialView.swift`
- Modify (shroomsweeper): `GameView.swift`, `TutorialView.swift`

**Interfaces:**
- Produces: `SegmentedToggle<Value: Hashable>(selection: Binding<Value>, segments: [SegmentedToggle<Value>.Segment])`, `Segment(_ value: Value, title: String, @ViewBuilder icon: () -> some View)`.

- [ ] **Step 1: Create the component**

`shroomkit/Sources/ShroomKit/Components/SegmentedToggle.swift`:
```swift
import SwiftUI

/// Generic pill-style segmented control. Each segment carries an arbitrary
/// leading glyph (custom shape or SF Symbol). Active segment gets the
/// `.isSelected` accessibility trait.
public struct SegmentedToggle<Value: Hashable>: View {
    public struct Segment: Identifiable {
        public let id: Value
        let title: String
        let icon: AnyView
        public init(_ value: Value, title: String, @ViewBuilder icon: () -> some View) {
            self.id = value
            self.title = title
            self.icon = AnyView(icon())
        }
    }

    @Binding private var selection: Value
    private let segments: [Segment]

    @Environment(\.palette) private var palette

    public init(selection: Binding<Value>, segments: [Segment]) {
        self._selection = selection
        self.segments = segments
    }

    public var body: some View {
        HStack(spacing: 6) {
            ForEach(segments) { segment in
                let isActive = segment.id == selection
                Button { selection = segment.id } label: {
                    HStack(spacing: 7) {
                        segment.icon
                        Text(segment.title)
                            .font(.system(.subheadline, design: .rounded).weight(.semibold))
                    }
                    .frame(maxWidth: .infinity)
                    .frame(minHeight: 44)
                    .padding(.vertical, 4)
                    .foregroundStyle(isActive ? palette.accentText : palette.sub)
                    .background(
                        RoundedRectangle(cornerRadius: Radius.md, style: .continuous)
                            .fill(isActive ? palette.accent : Color.clear)
                    )
                    .contentShape(Rectangle())
                }
                .buttonStyle(.plain)
                .accessibilityAddTraits(isActive ? .isSelected : [])
                .animation(.easeInOut(duration: 0.15), value: isActive)
            }
        }
        .padding(5)
        .background(
            RoundedRectangle(cornerRadius: Radius.xl, style: .continuous)
                .fill(palette.pill)
        )
    }
}
```

- [ ] **Step 2: Build ShroomKit** — `swift build`.

- [ ] **Step 3: Swap rootline** — in `PlayView.swift`, replace the `modeToggle`/`segment(...)` helpers with:
  ```swift
  SegmentedToggle(selection: $board.mode, segments: [
      .init(.draw, title: "Draw thread") { Capsule().fill(Color.primary).frame(width: 13, height: 3) },
      .init(.mark, title: "Mark dead") { Text("✕").font(.system(.footnote, design: .rounded).weight(.semibold)) },
  ])
  ```
  (Match the actual `Mode` enum cases + glyphs in the file.) Do the same swap in `TutorialView.swift` (the verbatim duplicate). Delete both local `modeToggle`/`segment` helpers. If `board.mode` isn't directly bindable, use `Binding(get: { board.mode }, set: { board.mode = $0 })`.

- [ ] **Step 4: Swap shroomsweeper** — in `GameView.swift` + `TutorialView.swift`, replace the duplicated mode toggle with `SegmentedToggle(selection:..., segments: [...])` using the game's reveal/flag enum + its SF-Symbol glyphs. Delete the local helpers.

- [ ] **Step 5: Compile-check** both apps → `BUILD SUCCEEDED`.

- [ ] **Step 6: Commit** (3 repos) — note this kills the toggle duplicated in 4 files.

- [ ] **Step 7: Review pause (Michelle)** — board mode toggle in both apps' play + tutorial screens; switching modes works; VoiceOver announces the selected segment.

---

## Task 4: ResultCard

**Files:**
- Create: `shroomkit/Sources/ShroomKit/Components/ResultCard.swift`
- Modify (rootline): `WinCard.swift`, `RevealedCard.swift`
- Modify (shroomsweeper): `ResultBar.swift`

**Interfaces:**
- Consumes: `ShroomButtonStyle` (`.shroomOutline`, `.shroomPrimary`), `palette`.
- Produces: `ResultCard<Badge: View>(title: String, subtitle: String, note: String? = nil, primaryLabel: String, onPrimary: @escaping () -> Void, secondaryLabel: String, onSecondary: @escaping () -> Void, @ViewBuilder badge: () -> Badge)`.

- [ ] **Step 1: Create the component**

`shroomkit/Sources/ShroomKit/Components/ResultCard.swift`:
```swift
import SwiftUI

/// End-of-session card: app-supplied badge + title/subtitle (+ optional accent
/// note) and a secondary/primary button pair.
public struct ResultCard<Badge: View>: View {
    private let title: String
    private let subtitle: String
    private let note: String?
    private let primaryLabel: String
    private let onPrimary: () -> Void
    private let secondaryLabel: String
    private let onSecondary: () -> Void
    private let badge: Badge

    @Environment(\.palette) private var palette

    public init(title: String, subtitle: String, note: String? = nil,
                primaryLabel: String, onPrimary: @escaping () -> Void,
                secondaryLabel: String, onSecondary: @escaping () -> Void,
                @ViewBuilder badge: () -> Badge) {
        self.title = title
        self.subtitle = subtitle
        self.note = note
        self.primaryLabel = primaryLabel
        self.onPrimary = onPrimary
        self.secondaryLabel = secondaryLabel
        self.onSecondary = onSecondary
        self.badge = badge()
    }

    public var body: some View {
        VStack(spacing: 14) {
            HStack(spacing: 12) {
                badge
                VStack(alignment: .leading, spacing: 2) {
                    Text(title)
                        .font(.system(.callout, design: .rounded).weight(.semibold))
                        .foregroundStyle(palette.text)
                    Text(subtitle)
                        .font(.system(.footnote, design: .rounded))
                        .foregroundStyle(palette.sub)
                    if let note {
                        Text(note)
                            .font(.system(.caption, design: .rounded).weight(.medium))
                            .foregroundStyle(palette.accent)
                    }
                }
                Spacer(minLength: 0)
            }
            HStack(spacing: 10) {
                Button(secondaryLabel, action: onSecondary).buttonStyle(.shroomOutline)
                Button(primaryLabel, action: onPrimary).buttonStyle(.shroomPrimary)
            }
        }
        .padding(.horizontal, 16)
        .padding(.vertical, 14)
        .background(
            RoundedRectangle(cornerRadius: Radius.xl, style: .continuous)
                .fill(palette.pill)
        )
    }
}
```

- [ ] **Step 2: Build ShroomKit** — `swift build`.

- [ ] **Step 3: Swap rootline `WinCard`** — replace the card body with:
  ```swift
  ResultCard(title: "Network connected!", subtitle: subtitle, note: fastestYet ? "Your fastest yet" : nil,
             primaryLabel: "Next puzzle", onPrimary: onNext,
             secondaryLabel: "Menu", onSecondary: onMenu) {
      RoundedRectangle(cornerRadius: Radius.md, style: .continuous)
          .fill(palette.tierSelBg)
          .frame(width: 44, height: 44)
          .overlay(Image(systemName: "checkmark").font(.system(.title3, design: .rounded).weight(.bold)).foregroundStyle(palette.accent))
  }
  ```
  Keep `WinCard`'s existing `subtitle`/`fastestYet` inputs and the outer `.padding`/transition the caller applies. Delete the inline card/button chrome.

- [ ] **Step 4: Swap rootline `RevealedCard`** — same shape, eye badge + its title/subtitle, `note: nil`.

- [ ] **Step 5: Swap shroomsweeper `ResultBar`** — `ResultCard(title:, subtitle:, primaryLabel: won ? "Play again" : "Try again", onPrimary: onPlayAgain, secondaryLabel: "Menu", onSecondary: onMenu) { MushroomIcon()... badge }`. Delete the inline chrome.

- [ ] **Step 6: Compile-check** both apps → `BUILD SUCCEEDED`.

- [ ] **Step 7: Commit** (3 repos).

- [ ] **Step 8: Review pause (Michelle)** — win/revealed cards (rootline) + result bar (shroomsweeper); the "Your fastest yet" note still appears on a new best; buttons work.

---

## Task 5: Settings chrome (SettingsSection + SelectionChip + SettingsRow)

**Files:**
- Create: `shroomkit/Sources/ShroomKit/Components/SettingsChrome.swift`
- Modify (rootline): `SettingsSheet.swift`
- Modify (shroomsweeper): `OptionsSheet.swift`

**Interfaces:**
- Consumes: `EyebrowLabel`, `palette`.
- Produces: `SettingsSection(_ title: String, @ViewBuilder content: () -> some View)`, `SelectionChip(_ label: String, isSelected: Bool, action: @escaping () -> Void)`, `SettingsRow(icon: String, label: String, action: @escaping () -> Void)`.

- [ ] **Step 1: Create the components**

`shroomkit/Sources/ShroomKit/Components/SettingsChrome.swift`:
```swift
import SwiftUI

/// Eyebrow-titled section wrapper for settings/options content.
public struct SettingsSection<Content: View>: View {
    private let title: String
    private let content: Content
    public init(_ title: String, @ViewBuilder content: () -> Content) {
        self.title = title
        self.content = content()
    }
    public var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            EyebrowLabel(title)
            content
        }
    }
}

/// Toggling chip for a single option among several.
public struct SelectionChip: View {
    private let label: String
    private let isSelected: Bool
    private let action: () -> Void
    @Environment(\.palette) private var palette
    public init(_ label: String, isSelected: Bool, action: @escaping () -> Void) {
        self.label = label
        self.isSelected = isSelected
        self.action = action
    }
    public var body: some View {
        Button(action: action) {
            Text(label)
                .font(.system(.subheadline, design: .rounded).weight(.semibold))
                .foregroundStyle(isSelected ? palette.accentText : palette.text)
                .padding(.horizontal, 16)
                .padding(.vertical, 10)
                .frame(minHeight: 44)
                .background(
                    RoundedRectangle(cornerRadius: Radius.md, style: .continuous)
                        .fill(isSelected ? palette.accent : palette.pill)
                )
                .contentShape(Rectangle())
        }
        .buttonStyle(.plain)
        .accessibilityAddTraits(isSelected ? .isSelected : [])
    }
}

/// Tappable settings row: leading accent icon + label + trailing chevron.
public struct SettingsRow: View {
    private let icon: String
    private let label: String
    private let action: () -> Void
    @Environment(\.palette) private var palette
    public init(icon: String, label: String, action: @escaping () -> Void) {
        self.icon = icon
        self.label = label
        self.action = action
    }
    public var body: some View {
        Button(action: action) {
            HStack {
                Image(systemName: icon)
                    .font(.system(.subheadline, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.accent)
                Text(label)
                    .font(.system(.subheadline, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.text)
                Spacer()
                Image(systemName: "chevron.right")
                    .font(.system(.caption, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.sub)
            }
            .padding(.horizontal, 14)
            .padding(.vertical, 12)
            .frame(minHeight: 44)
            .background(
                RoundedRectangle(cornerRadius: Radius.lg, style: .continuous)
                    .fill(palette.tierBg)
            )
            .contentShape(Rectangle())
        }
        .buttonStyle(.plain)
    }
}
```

- [ ] **Step 2: Build ShroomKit** — `swift build`.

- [ ] **Step 3: Swap rootline `SettingsSheet`** — replace the local `section(title:)` with `SettingsSection(title) { ... }`; the `chip(label:isActive:)` with `SelectionChip(label, isSelected: ...) { ... }`; the two inline tappable rows ("Replay tutorial lessons", DEBUG "Puzzle editor") with `SettingsRow(icon:, label:) { ... }`. Delete the local helpers.

- [ ] **Step 4: Swap shroomsweeper `OptionsSheet`** — section headers → `SettingsSection`; theme/look chips → `SelectionChip`. (If a setting uses a native `Toggle`, leave it.) Delete the replaced local helpers.

- [ ] **Step 5: Compile-check** both apps → `BUILD SUCCEEDED`.

- [ ] **Step 6: Commit** (3 repos).

- [ ] **Step 7: Review pause (Michelle)** — Settings/Options sheets in both apps; section headers, chips (selected state + VoiceOver), and rows look/behave unchanged.

---

## Notes for the implementer

- **Read each call site before swapping** (line numbers drift); match the actual enum cases, property names, and closures in the file. The harness's "No such module 'ShroomKit'" / "Cannot find Radius" SourceKit diagnostics are stale single-file noise — `swift build` + `xcodebuild` are the truth.
- **Bindings:** `SegmentedToggle` needs a `Binding`. If a board/game `mode` isn't exposed as a SwiftUI binding, wrap with `Binding(get:set:)`.
- **Delete local helpers** after each swap so the call site is the only definition — no shims.
- **If a swap feels awkward** (the ShroomKit validation rule), stop and report — adjust the API rather than work around it at the call site. (ResultCard especially: if WinCard/ResultBar don't fit cleanly, the badge slot or note param may need rethinking.)
- **`.accessibilityAddTraits(isSelected ? .isSelected : [])`** — passing an empty `AccessibilityTraits` ([]) when not selected is valid and a no-op.
- xcodebuild is slow (~1–2 min/app); run once per task after both apps are swapped.
