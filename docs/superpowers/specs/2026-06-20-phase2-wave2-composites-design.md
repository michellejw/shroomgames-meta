# Phase 2 Wave 2 — Composite Components (Design Spec)

**Date:** 2026-06-20
**Part of:** Big Rock #2, Phase 2 (iOS component consolidation). Wave 2 of 3 (audit: `2026-06-19-phase2-component-audit.md`; Wave 1: `2026-06-19-phase2-wave1-primitives-design.md`).
**Depends on:** ShroomKit Wave 1 primitives (`PillIconButton`, `EyebrowLabel`, `ShroomButtonStyle`) + tokens.

## Goal

Extract the five composite UI patterns shared across rootline + shroomsweeper into ShroomKit, composing the Wave 1 primitives, with accessibility fixed (selection traits) and tokens adopted. Swap both apps onto them.

## Conventions (carried from Wave 1, unchanged)
- Views read `@Environment(\.palette)`; use `Radius`/`FontTracking` tokens *inside the components only*.
- Accessibility baked in on extraction. Tap targets ≥ 44pt.
- No back-compat shims; update kit + both apps together; delete replaced local helpers.
- Build gate: ShroomKit `swift build`; app compile via `xcodebuild build -scheme <app> -destination 'generic/platform=iOS Simulator'`. Visual confirmation is a per-component review pause (Michelle).

## Components

### 1. `ScreenHeader` — `Components/ScreenHeader.swift`
**Slot-based and deliberately flexible** — the component owns the skeleton (back + title + trailing area); each screen supplies its own trailing buttons. Built this way so a *game* header (centered two-line title + multiple action buttons) could adopt it later via a small swap, not a rewrite — but game headers are NOT adopted this wave (see below).
```swift
public struct ScreenHeader<Trailing: View>: View {
    public init(_ title: String, subtitle: String? = nil, onBack: (() -> Void)?,
                @ViewBuilder trailing: () -> Trailing = { EmptyView() })
}
```
- HStack: a back `PillIconButton(systemName: "chevron.left", accessibilityLabel: "Back")` when `onBack != nil`; the title in `.title2` rounded semibold (`palette.text`) with an optional `subtitle` as an `EyebrowLabel` above it (the future game-header two-line shape); `Spacer`; the `trailing` slot.
- **Adopt this wave:** rootline `DifficultyView.header`, `StatsView.header` (trailing = the "Clear" ghost button); shroomsweeper's simple screen headers.
- **Deferred (not this wave):** rootline `PlayView.header` and shroomsweeper `GameView.header` stay bespoke. Their *actions* differ because the games differ (slitherlink hints/reveal vs minesweeper), so whether the game headers should structurally match or look identical is a product call to make later. The slot-based API means adopting them later is a swap, not a rewrite. (Their buttons already use the shared `PillIconButton`, so they aren't un-consolidated.)

### 2. `SegmentedToggle` — `Components/SegmentedToggle.swift`
Generic 2-segment (or N) pill toggle bound to a value. Per-segment glyph is an arbitrary view (rootline's are custom shapes, not SF Symbols).
```swift
public struct SegmentedToggle<Value: Hashable>: View {
    public struct Segment: Identifiable {
        public let id: Value
        let title: String
        let icon: AnyView
        public init(_ value: Value, title: String, @ViewBuilder icon: () -> some View)
    }
    public init(selection: Binding<Value>, segments: [Segment])
}
```
- Outer pill container (`palette.pill`, `Radius.xl`, inner padding); each segment is a `Button` (`minHeight: 44`): active = `palette.accent` fill + `palette.accentText`; inactive = `Color.clear` + `palette.sub`; inner `Radius.md`. The active segment gets `.accessibilityAddTraits(.isSelected)` (the audit's missing trait). Tapping a segment sets `selection`.
- **Replaces:** the verbatim-duplicated `modeToggle`/`segment` in rootline `PlayView` + `TutorialView`, and shroomsweeper `GameView` + `TutorialView`. rootline binds `$board.mode` (draw/mark) with a thread-capsule + ✕ glyph; shroomsweeper binds its reveal/flag mode with SF Symbols.

### 3. `SelectionCard` — `Components/SelectionCard.swift`
```swift
public struct SelectionCard: View {
    public init(title: String, subtitle: String, isSelected: Bool, action: @escaping () -> Void)
}
```
- Tappable card: leading title (`.headline`) + subtitle (`.subheadline`, `palette.sub`) VStack; `Spacer`; trailing radio dot (outer `Circle` stroke `palette.accent`/`palette.tierBorder` lineWidth 2; inner filled `Circle` when selected). Card fill `palette.tierSelBg`/`palette.tierBg` + border, `Radius.xl`. `.accessibilityAddTraits(.isSelected)` when selected; whole card is the button, ≥44pt.
- **Replaces:** rootline `DifficultyView.tierRow`, shroomsweeper `OptionsSheet.difficultyRow`.

### 4. `ResultCard` — `Components/ResultCard.swift`
The richest; composes Wave 1 button styles.
```swift
public struct ResultCard<Badge: View>: View {
    public init(title: String, subtitle: String, note: String? = nil,
                primaryLabel: String, onPrimary: @escaping () -> Void,
                secondaryLabel: String, onSecondary: @escaping () -> Void,
                @ViewBuilder badge: () -> Badge)
}
```
- `palette.pill` card (`Radius.xl`): top HStack of `badge` slot (app-supplies the checkmark tile / eye / mascot) + title (`.callout`/`.headline` semibold) + subtitle (`.footnote`, `palette.sub`) + optional `note` line (`.caption`, `palette.accent` — the "Your fastest yet" whisper); bottom button row: `Button(secondaryLabel, action: onSecondary).buttonStyle(.shroomOutline)` + `Button(primaryLabel, action: onPrimary).buttonStyle(.shroomPrimary)`.
- **Replaces:** rootline `WinCard` + `RevealedCard`, shroomsweeper `ResultBar`. (shroomsweeper's `WinEntryCard` is the arcade initials-entry card — leave it; not a result card.)

### 5. Settings chrome — `Components/SettingsChrome.swift` (three small types in one file)
```swift
public struct SettingsSection<Content: View>: View {           // EyebrowLabel(title) + content
    public init(_ title: String, @ViewBuilder content: () -> Content)
}
public struct SelectionChip: View {                            // toggling chip
    public init(_ label: String, isSelected: Bool, action: @escaping () -> Void)
}
public struct SettingsRow: View {                              // icon + label + chevron, tappable
    public init(icon: String, label: String, action: @escaping () -> Void)
}
```
- `SettingsSection`: `VStack(alignment: .leading)` of `EyebrowLabel(title)` + `content()`.
- `SelectionChip`: pill button, active `palette.accent`/`accentText`, inactive `palette.pill`/`text`, `Radius.md`, ≥44pt, `.accessibilityAddTraits(.isSelected)` when active.
- `SettingsRow`: accent icon + `.subheadline` label + `Spacer` + `chevron.right` (`palette.sub`) in a `palette.tierBg` rounded row (`Radius.lg`), ≥44pt, whole row a button.
- **Replaces:** rootline `SettingsSheet` (`section`/`chip` helpers + the two inline rows), shroomsweeper `OptionsSheet` (section headers, theme/look chips).

## Testing
- ShroomKit `swift build` (headless). No new pure logic to unit-test (these are SwiftUI views); the existing token/icon tests stay green.
- App compile via `xcodebuild build` per component swap.
- Per-component visual review pause (Michelle): confirm each screen unchanged except intended standardizations, and that VoiceOver now announces selection state on toggles/cards/chips.

## Out of scope
- Wave 3 (tutorial scaffold: `TutorialBannerCard` + `NudgeToast`).
- rootline `PlayView` bespoke header; shroomsweeper `WinEntryCard` arcade entry.
- App-wide literal sweep; mascots; game logic.
- The sheet's tabbed tab-bar (shroomsweeper `OptionsSheet`) — single-app, defer unless rootline grows one.

## Open questions (decide during planning)
- `SegmentedToggle` segment count: build for N segments (array) even though both apps use 2 — yes, array is barely more code and future-proofs.
- `ResultCard` badge default: require the slot (no default) — both apps always supply one.
