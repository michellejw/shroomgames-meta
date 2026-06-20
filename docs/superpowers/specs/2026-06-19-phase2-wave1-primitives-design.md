# Phase 2 Wave 1 — Shared UI Primitives (Design Spec)

**Date:** 2026-06-19
**Part of:** Big Rock #2, Phase 2 (iOS component consolidation). Wave 1 of 3 (see the audit: `2026-06-19-phase2-component-audit.md`).
**Depends on:** ShroomKit's existing Palette + `Radius`/`Space`/`FontTracking` tokens (Phase 1).

## Goal

Extract the four most-duplicated, lowest-risk UI primitives from rootline + shroomsweeper into ShroomKit, fix their accessibility once, and have them consume the design tokens — then swap both apps onto them. This proves the "extract → swap both apps → does it feel natural?" loop before the larger composites in Waves 2–3.

## Decisions (settled)
- Buttons are SwiftUI **`ButtonStyle`s**; the icon button, label, and stat pill are **Views**.
- **Accessibility fixes baked in** on extraction (required labels, selection where relevant, 44pt floor, `.caption2`→`.caption` eyebrow).
- **Token adoption only inside the extracted components** — no app-wide literal sweep this wave.
- Per ShroomKit convention: no back-compat shims; update kit + both apps together; review-pause per component.

## Components

### 1. Button styles — `Sources/ShroomKit/Components/ShroomButtonStyle.swift`

One `ButtonStyle` struct with a `kind` and a `prominent` flag, exposed via static accessors:
```swift
public struct ShroomButtonStyle: ButtonStyle {
    public enum Kind { case primary, secondary, outline }
    let kind: Kind
    let prominent: Bool
    // makeBody: rounded-semibold font (.headline, or .title3 when prominent),
    // foreground + background per kind, frame(maxWidth: .infinity, minHeight: 44),
    // Radius.xl corner, pressed-state (opacity ~0.9 + scale ~0.98 when isPressed).
}
public extension ButtonStyle where Self == ShroomButtonStyle {
    static var shroomPrimary: ShroomButtonStyle { .init(kind: .primary, prominent: false) }
    static func shroomPrimary(prominent: Bool) -> ShroomButtonStyle { .init(kind: .primary, prominent: prominent) }
    static var shroomSecondary: ShroomButtonStyle { .init(kind: .secondary, prominent: false) }
    static var shroomOutline: ShroomButtonStyle { .init(kind: .outline, prominent: false) }
}
```
- **primary:** `palette.accent` fill, `palette.accentText` foreground.
- **secondary:** `palette.pill` fill, `palette.text` foreground.
- **outline:** `palette.tierBg` fill + `palette.tierBorder` 1pt stroke, `palette.text` foreground.
- All: `Radius.xl` (16) — unifies the current 14/16/18 spread; `minHeight: 44`; rounded-semibold; pressed-state dim/scale. Reads palette from `@Environment(\.palette)` (a `ButtonStyle` can use `@Environment`).
- Font: `.headline` default, `.title3` when `prominent` (covers the big Home "Play").

**Replaces:** rootline `HomeView.primaryButton`, `TutorialView` CTA, the result-card button pair (Wave 2 will reuse outline+primary here); shroomsweeper `HomeView`, `OptionsSheet`, `TutorialView`, `ResultBar`, `WinEntryCard` inline buttons. `WelcomeScaffold`'s inline primary/secondary get swapped to these too.

### 2. `PillIconButton` — `Sources/ShroomKit/Components/PillIconButton.swift`
```swift
public struct PillIconButton: View {
    public enum Shape { case roundedRect, circle }
    public init(systemName: String, accessibilityLabel: String,
                shape: Shape = .roundedRect, isEnabled: Bool = true,
                action: @escaping () -> Void)
}
```
- 44×44; `palette.pill` background; `palette.sub` foreground (≈0.35 opacity when `!isEnabled`); `Radius.md` for `roundedRect`, `Circle` for `circle`; `.contentShape` matching the visual; rounded-semibold body icon.
- **`accessibilityLabel` is required** (the a11y fix). Applies `.accessibilityLabel(...)` and, when disabled, the button is `.disabled(true)`.

**Replaces:** every toolbar icon button — rootline `PlayView` (back/eye/theme/hint ×4), `DifficultyView`/`StatsView` back, `HomeView` settings gear; shroomsweeper `HomeView`/`WelcomeView`/`GameView` icon buttons. The duplicated `themeIconName(for:)` helper moves into ShroomKit alongside (it maps `ThemeMode` → SF Symbol, which is ShroomKit's type).

### 3. `EyebrowLabel` — `Sources/ShroomKit/Components/EyebrowLabel.swift`
```swift
public struct EyebrowLabel: View {
    public init(_ text: String, tint: EyebrowTint = .sub)  // .sub | .accent
}
```
- Renders `text` with `.textCase(.uppercase)` (call with sentence case: `EyebrowLabel("Difficulty")`); `.caption` rounded semibold (bumped from `.caption2` for legibility); `FontTracking.eyebrow` tracking; foreground `palette.sub` or `palette.accent` per `tint`.

**Replaces:** rootline `HomeView` "DIFFICULTY", `PlayView` tier label, `StatsView` tier labels, `SettingsSheet.section` title; shroomsweeper `HomeView`, `TutorialView` step label (accent tint). Call sites change from uppercased string literals to sentence-case.

### 4. `StatPill` — `Sources/ShroomKit/Components/StatPill.swift`
```swift
public struct StatPill: View {
    public init(_ text: String, systemName: String? = nil)
}
```
- Optional leading SF Symbol (`palette.sub`) + `text` in rounded-semibold with `.monospacedDigit()` (`palette.text`); `palette.pill` background; `Radius.md`; non-interactive.

**Replaces:** rootline `PlayView.statPill` (timer); shroomsweeper `GameView.statPill`. (The rootline "hints" pill uses a dot indicator rather than an SF Symbol — out of scope for the convenience init; it stays app-local this wave, or gets a leading-slot init in a later pass. Note, don't force it.)

## Accessibility (baked in)
- `PillIconButton`: required `accessibilityLabel`.
- `EyebrowLabel`: `.caption` (not `.caption2`); `.textCase(.uppercase)` on sentence-case source.
- Buttons: guaranteed `minHeight: 44`.
- (Selection `.isSelected` traits belong to Wave 2's toggle/selection components, not here.)

## Token usage
Inside these components only: `Radius.xl`/`Radius.md`, `FontTracking.eyebrow`. Spacing values inside components use `Space.*` where a clean match exists; otherwise keep the existing literal (don't invent tokens). No literal sweep outside the components.

## Testing
- **ShroomKit:** `swift build` (headless) confirms the package compiles. Swift Testing has little to assert on pure-SwiftUI views; if any non-view helper is added (e.g. `themeIconName`), unit-test it. No snapshot framework is added (scope).
- **App swaps (Xcode, GUI-bound — Michelle):** after each component swap, build rootline + shroomsweeper and visually confirm the screen is unchanged except the intended standardizations (unified button radius, slightly larger eyebrow). This is the real validation and the "does the API feel natural?" check.

## Out of scope
- Wave 2 composites (ScreenHeader, SegmentedToggle, SelectionCard, ResultCard, settings chrome) and Wave 3 (tutorial flow).
- App-wide literal sweep; mascots/icons; game logic.
- The rootline hints-pill dot variant (stays app-local this wave).

## Open questions (decide during planning)
- Whether the button `prominent` font should be `.title3` or keep `.headline` everywhere (lean `.title3` for Home "Play" only).
- `EyebrowTint` as an enum vs a raw `Color` param — lean enum (`.sub`/`.accent`) to keep call sites token-bound.
