# Phase 2 — Wave 3: Tutorial scaffold (`TutorialBannerCard` + `NudgeToast`)

**Date:** 2026-06-20
**Part of:** Big Rock #2 — design system, Phase 2 (iOS component consolidation), Wave 3 (final wave).
**Status:** Design approved by Michelle. Spec → plan → execute next.

## Goal

Extract the two remaining shared tutorial-chrome patterns from rootline's and
shroomsweeper's `TutorialView` into ShroomKit, fixing the twilight-shadow a11y
bug on the way. Flow state (`TutorialFlow`, lesson content, highlighting,
nudge timing) stays app-side — the kit owns only the chrome.

This closes Phase 2 (Waves 1 primitives + 2 composites already merged).

## Why these two are the hard wave

The two apps' tutorial chrome diverges more than earlier waves' patterns did,
so the unification is deliberately pragmatic:

- **Banners** share a "rounded card wrapping tutorial text" skeleton but differ
  in which slots they use (shroomsweeper: eyebrow + inline Skip + inline CTA;
  rootline: title + body only — its Skip lives in a separate top bar and its
  CTA in the post-solve unlock strip).
- **Nudges** differ mostly in *layout*, which is the app's job, not the
  component's: shroomsweeper overlays the board and slides in; rootline pins a
  fixed-height inline slot to stop layout jumping. What's genuinely shared is
  the **pill itself**.

Decision: unify the banner as a slot-based card; share only the nudge *pill*
and keep placement/transition app-local. This avoids both a single-consumer
component and a bloated mega-API.

## Component 1 — `TutorialBannerCard`

`Sources/ShroomKit/Components/TutorialBannerCard.swift`

```swift
public struct TutorialBannerCard<Trailing: View, Footer: View>: View {
    public init(
        eyebrow: String? = nil,          // rendered via EyebrowLabel
        title: String,
        message: String,
        @ViewBuilder trailing: () -> Trailing = { EmptyView() },  // e.g. Skip
        @ViewBuilder footer: () -> Footer = { EmptyView() }       // e.g. CTA
    )
}
```

- **Container:** `RoundedRectangle(cornerRadius: Radius.xxl /* 18 */, style: .continuous)`
  filled `palette.pill`; horizontal pad `Space.md` (16), vertical pad 14
  (matches `ResultCard`). Reads color from `@Environment(\.palette)`.
- **Header row** (`HStack { EyebrowLabel(eyebrow); Spacer(); trailing() }`)
  renders **only when `eyebrow != nil`** — keeps rootline's plain card
  gap-free. Documented constraint: the trailing slot rides with the eyebrow.
  The only real trailing use (shroomsweeper's Skip) always co-occurs with an
  eyebrow, so this is sufficient; revisit if a future app needs trailing
  without an eyebrow.
- **Type ramp (unified — Michelle's call):** title
  `.system(.title3, design: .rounded).weight(.semibold)`, message
  `.system(.callout, design: .rounded)` in `palette.sub`, `lineSpacing(2)`,
  `fixedSize(horizontal: false, vertical: true)`. rootline's banner grows
  slightly from its current `.subheadline`/`.footnote`; confirm at review.
- **Footer:** rendered below the message; the call site supplies its own top
  padding (as shroomsweeper already does). Empty by default.

### Consumers

- **rootline** (`instructionPill` → replace):
  `TutorialBannerCard(title: flow.lesson.title, message: flow.lesson.instruction)`
  — slots empty. Its "Skip all / Skip lesson" top bar and post-solve unlock
  strip CTA stay where they are.
- **shroomsweeper** (`tutorialBanner` → replace): all slots lit —
  `eyebrow: flow.stepLabel`, `trailing:` the Skip button, `footer:` the
  "Got it" / "Start foraging" CTA (whichever `flow` flags).

## Component 2 — `NudgeToast`

`Sources/ShroomKit/Components/NudgeToast.swift`

```swift
public enum NudgeTone { case guidance, warning }   // accent vs warn

public struct NudgeToast: View {
    public init(_ message: String, tone: NudgeTone = .guidance)
}
```

- **One look (Michelle's call):** `RoundedRectangle(cornerRadius: Radius.md /* 12 */)`
  filled `palette.tierSelBg`, tone-tinted 1.5pt border
  (`.guidance` → `palette.accent`, `.warning` → `palette.warn`).
- **Content:** leading icon per tone (`.guidance` → `lightbulb.fill`,
  `.warning` → `exclamationmark.circle.fill`) tinted to the tone color, then
  the message in `palette.text`, leading-aligned, `fixedSize` vertical so it
  wraps. shroomsweeper's nudge gains a leading lightbulb under this look.
- **No shadow.** Drops shroomsweeper's hardcoded `.black.opacity(0.18)` drop
  shadow — invisible on twilight's dark bg (the audited bug). The tone-tinted
  border carries float definition in both themes, so no replacement shadow and
  no new token is needed (keeps Wave 3 out of the token pipeline).
- **The component is just the pill** — no placement, no transition. Each app
  supplies those:
  - **shroomsweeper** overlays the board with its existing
    `.move(edge: .top).combined(with: .opacity)` transition.
  - **rootline** drops it into its existing fixed-height, opacity-toggled
    coaching slot (the anti-jump frame stays local); maps its over-fill error
    → `.warning`, its stuck-hint → `.guidance`. The over-fill / closed-loop /
    stuck-hint logic in `TutorialView` is untouched.

## Accessibility (baked in on extraction)

- `TutorialBannerCard` trailing Skip and footer CTA keep the 44pt floor
  (CTA via `.shroomPrimary` already complies; the app-supplied Skip button must
  too — its existing one already uses `minHeight: 44`).
- `NudgeToast` sets `.accessibilityElement(children: .combine)` with an
  `.accessibilityLabel` that prefixes the tone (e.g. `"Warning: \(message)"` /
  `"Tip: \(message)"`) so VoiceOver conveys what the color + icon imply.
- Eyebrow renders via `EyebrowLabel` (already `.caption`, large-Dynamic-Type
  safe).

## Out of scope

- Flow state machines (`TutorialFlow`), lesson/step content, target
  highlighting, nudge timing — all stay app-side.
- App mascots/icons (`MyceliumIcon`, `MushroomIcon`, `FlagIcon`) — consumer
  supplies via the slots / wrappers.
- rootline's fixed-height anti-jump coaching slot and shroomsweeper's
  board-overlay transition — layout, stays app-side.
- A shadow/elevation token in the token pipeline — deliberately deferred;
  border-only float was chosen instead.

## Verification

- **ShroomKit:** `swift build`; `swift test` — add render/smoke tests for both
  components and the `NudgeTone` → icon/color mapping where sensible.
- **Both apps:**
  `xcodebuild build -project <app>.xcodeproj -scheme <app> -destination 'generic/platform=iOS Simulator'`.
- **Workflow:** inline, per-component review pause — build `TutorialBannerCard`
  in ShroomKit, swap **both** apps, Michelle reviews; then `NudgeToast`, swap
  both apps, Michelle reviews. If a swap feels awkward, the API isn't done
  (per ShroomKit CLAUDE.md). In shroomsweeper, stage only edited files
  (`git add <files>`, never `-A`). Git: branch off main, FF-merge, push,
  delete branch, in all three repos; Claude co-author trailer on commits.
```
