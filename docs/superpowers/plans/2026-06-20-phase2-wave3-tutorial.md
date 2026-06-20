# Phase 2 Wave 3 — Tutorial scaffold Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Note (Michelle's workflow):** iOS component work is driven **inline** with a **visual review pause after each component**. This plan is two component arcs (NudgeToast, then TutorialBannerCard); each arc = build in ShroomKit → swap both apps → Michelle reviews → merge all repos.

**Goal:** Extract the two remaining shared tutorial-chrome patterns (`NudgeToast`, `TutorialBannerCard`) into ShroomKit, fixing the twilight-invisible drop-shadow, and adopt them in rootline + shroomsweeper. Closes Phase 2.

**Architecture:** Two `@Environment(\.palette)`-driven SwiftUI views in `ShroomKit/Components`. The kit owns only chrome; flow state, placement, and transitions stay app-side. `NudgeTone` carries a small testable icon/accessibility mapping (mirrors the existing `ThemeMode.iconName` test pattern); the views are verified by `swift build` + downstream app builds + visual review (no view-render unit tests — not the house pattern).

**Tech Stack:** SwiftUI (iOS 17+), Swift Testing (`import Testing`), ShroomKit local-path SwiftPM dependency, Style-Dictionary tokens (`Radius`/`Space`).

## Global Constraints

- iOS 17+ minimum; SwiftUI only (no UIKit/Combine); `@Observable` not `ObservableObject`; 4-space indent.
- Views never hard-code colors — read from `@Environment(\.palette)`. Use `Radius`/`Space` tokens, not literals, for the values that have tokens (`Radius.md` = 12, `Radius.xxl` = 18, `Space.md` = 16).
- 44pt minimum tap targets (Michelle runs large Dynamic Type).
- Public API gets one-line doc comments only where the WHY is non-obvious. No back-compat shims (two consumers, update together).
- ShroomKit verifies headless: `swift build` / `swift test` from `shroomkit/`.
- Apps verify with: `xcodebuild build -project <app>.xcodeproj -scheme <app> -destination 'generic/platform=iOS Simulator'` (named 'iPhone 16' destination is flaky — use generic). Ignore harness SourceKit "No such module 'ShroomKit'" noise; the CLI builds are the truth.
- Apps consume ShroomKit via local path `../shroomkit`, so they compile against shroomkit's **current working-tree checkout** — keep the kit feature branch checked out while building the apps.
- Git per repo: branch off `main`, commit per step, FF-merge + push + delete branch **after** the review gate. End commit messages with:
  `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`
- In **shroomsweeper**, stage only the files you edited (`git add <files>`, never `-A`) — its pbxproj has churn history.

---

## Component arc A — NudgeToast

### Task 1: `NudgeToast` + `NudgeTone` in ShroomKit

**Files:**
- Create: `shroomkit/Sources/ShroomKit/Components/NudgeToast.swift`
- Create: `shroomkit/Tests/ShroomKitTests/NudgeToneTests.swift`

**Interfaces:**
- Produces:
  - `public enum NudgeTone { case guidance, warning }` with `var iconName: String` and `var accessibilityPrefix: String`.
  - `public struct NudgeToast: View` with `public init(_ message: String, tone: NudgeTone = .guidance)`.

- [ ] **Step 1: Branch**

```bash
cd ~/dev/games/shroom-games/shroomkit && git checkout main && git checkout -b feature/wave3-nudgetoast
```

- [ ] **Step 2: Write the failing test**

Create `shroomkit/Tests/ShroomKitTests/NudgeToneTests.swift`:

```swift
import Testing
@testable import ShroomKit

struct NudgeToneTests {
    @Test func iconNamesAreStable() {
        #expect(NudgeTone.guidance.iconName == "lightbulb.fill")
        #expect(NudgeTone.warning.iconName == "exclamationmark.circle.fill")
    }

    @Test func accessibilityPrefixesAreStable() {
        #expect(NudgeTone.guidance.accessibilityPrefix == "Tip")
        #expect(NudgeTone.warning.accessibilityPrefix == "Warning")
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd ~/dev/games/shroom-games/shroomkit && swift test 2>&1 | tail -20`
Expected: FAIL — `cannot find 'NudgeTone' in scope`.

- [ ] **Step 4: Write `NudgeToast.swift`**

Create `shroomkit/Sources/ShroomKit/Components/NudgeToast.swift`:

```swift
import SwiftUI

/// The tone of a tutorial nudge — drives icon, tint, and the VoiceOver prefix.
public enum NudgeTone {
    case guidance
    case warning

    var iconName: String {
        switch self {
        case .guidance: "lightbulb.fill"
        case .warning: "exclamationmark.circle.fill"
        }
    }

    var accessibilityPrefix: String {
        switch self {
        case .guidance: "Tip"
        case .warning: "Warning"
        }
    }
}

/// A transient tutorial coaching pill: leading tone icon + message inside a
/// tone-tinted border. Placement and transition are the consumer's job — drop
/// this into a board overlay or a fixed-height slot as the flow requires.
/// No drop-shadow: the tinted border carries float definition in both themes
/// (a hardcoded `.black` shadow was invisible on twilight's dark background).
public struct NudgeToast: View {
    private let message: String
    private let tone: NudgeTone

    @Environment(\.palette) private var palette

    public init(_ message: String, tone: NudgeTone = .guidance) {
        self.message = message
        self.tone = tone
    }

    private var tint: Color {
        switch tone {
        case .guidance: palette.accent
        case .warning: palette.warn
        }
    }

    public var body: some View {
        HStack(alignment: .top, spacing: 10) {
            Image(systemName: tone.iconName)
                .font(.system(.footnote, design: .rounded).weight(.semibold))
                .foregroundStyle(tint)
            Text(message)
                .font(.system(.subheadline, design: .rounded).weight(.semibold))
                .foregroundStyle(palette.text)
                .fixedSize(horizontal: false, vertical: true)
            Spacer(minLength: 0)
        }
        .padding(.horizontal, 13)
        .padding(.vertical, 10)
        .frame(maxWidth: .infinity, alignment: .leading)
        .background(
            RoundedRectangle(cornerRadius: Radius.md, style: .continuous)
                .fill(palette.tierSelBg)
                .overlay(
                    RoundedRectangle(cornerRadius: Radius.md, style: .continuous)
                        .stroke(tint, lineWidth: 1.5)
                )
        )
        .accessibilityElement(children: .combine)
        .accessibilityLabel("\(tone.accessibilityPrefix): \(message)")
    }
}
```

- [ ] **Step 5: Run build + tests to verify they pass**

Run: `cd ~/dev/games/shroom-games/shroomkit && swift build 2>&1 | tail -5 && swift test 2>&1 | tail -20`
Expected: build succeeds; `NudgeToneTests` pass; no other test regressions.

- [ ] **Step 6: Commit (no merge yet — gated on Task 2 review)**

```bash
cd ~/dev/games/shroom-games/shroomkit
git add Sources/ShroomKit/Components/NudgeToast.swift Tests/ShroomKitTests/NudgeToneTests.swift
git commit -m "feat: add NudgeToast + NudgeTone tutorial coaching pill

Tone-tinted border (no drop-shadow — the hardcoded .black shadow was
invisible on twilight). Placement/transition stay consumer-side.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Adopt `NudgeToast` in rootline + shroomsweeper

**Files:**
- Modify: `rootline/rootline/Views/TutorialView.swift` (the `coachingSlot` computed property, ~lines 158-183)
- Modify: `shroomsweeper/shroomsweeper/Views/TutorialView.swift` (the board-overlay use ~lines 33-38 + delete the `nudgeToast(_:)` func ~lines 118-135)

**Interfaces:**
- Consumes: `NudgeToast(_ message: String, tone: NudgeTone)` from Task 1.

- [ ] **Step 1: Branch both apps** (keep `shroomkit` on `feature/wave3-nudgetoast`)

```bash
cd ~/dev/games/shroom-games/rootline && git checkout main && git checkout -b feature/wave3-nudgetoast
cd ~/dev/games/shroom-games/shroomsweeper && git checkout main && git checkout -b feature/wave3-nudgetoast
```

- [ ] **Step 2: Replace rootline's `coachingSlot`**

In `rootline/rootline/Views/TutorialView.swift`, replace the entire `coachingSlot` computed property with:

```swift
    // MARK: Coaching slot (fixed height, opacity-toggled so layout never jumps)

    private var coachingSlot: some View {
        let msg = errorMessage ?? stuckHint
        let tone: NudgeTone = errorMessage != nil ? .warning : .guidance
        return Group {
            if let msg {
                NudgeToast(msg, tone: tone)
                    .lineLimit(2)
            } else {
                Color.clear
            }
        }
        .frame(maxWidth: .infinity, alignment: .leading)
        .frame(minHeight: coachingHeight)
        .opacity(msg == nil ? 0 : 1)
        .animation(.easeInOut(duration: 0.2), value: errorMessage)
        .animation(.easeInOut(duration: 0.2), value: stuckHint)
    }
```

(The fixed `coachingHeight` frame + opacity toggle — rootline's anti-jump behaviour — stays local. `.lineLimit(2)` propagates to `NudgeToast`'s `Text`, preserving the old 2-line cap. The error→`.warning` / hint→`.guidance` mapping replaces the old `isError` icon/color switch.)

- [ ] **Step 3: Replace shroomsweeper's board-overlay nudge + delete its local func**

In `shroomsweeper/shroomsweeper/Views/TutorialView.swift`, in `body`, replace:

```swift
                if let msg = flow.nudgeMessage {
                    nudgeToast(msg)
                        .padding(.top, 8)
                        .padding(.horizontal, 12)
                        .transition(.move(edge: .top).combined(with: .opacity))
                }
```

with:

```swift
                if let msg = flow.nudgeMessage {
                    NudgeToast(msg)
                        .padding(.top, 8)
                        .padding(.horizontal, 12)
                        .transition(.move(edge: .top).combined(with: .opacity))
                }
```

Then delete the now-unused private `nudgeToast(_:)` function entirely:

```swift
    private func nudgeToast(_ msg: String) -> some View {
        Text(msg)
            .font(.system(.subheadline, design: .rounded).weight(.semibold))
            .foregroundStyle(palette.accent)
            .multilineTextAlignment(.center)
            .padding(.horizontal, 13)
            .padding(.vertical, 10)
            .frame(maxWidth: .infinity)
            .background(
                RoundedRectangle(cornerRadius: 12, style: .continuous)
                    .fill(palette.tierBg)
                    .overlay(
                        RoundedRectangle(cornerRadius: 12, style: .continuous)
                            .stroke(palette.accent, lineWidth: 1.5)
                    )
            )
            .shadow(color: .black.opacity(0.18), radius: 10, x: 0, y: 6)
    }
```

(The overlay placement + slide-in transition stay local. shroomsweeper's nudges are guidance-only, so default tone. The deleted func is where the twilight-invisible `.black` shadow lived.)

- [ ] **Step 4: Build both apps**

```bash
cd ~/dev/games/shroom-games/rootline && xcodebuild build -project rootline.xcodeproj -scheme rootline -destination 'generic/platform=iOS Simulator' 2>&1 | tail -5
cd ~/dev/games/shroom-games/shroomsweeper && xcodebuild build -project shroomsweeper.xcodeproj -scheme shroomsweeper -destination 'generic/platform=iOS Simulator' 2>&1 | tail -5
```
Expected: `** BUILD SUCCEEDED **` for both.

- [ ] **Step 5: REVIEW PAUSE**

Stop. Ask Michelle to review NudgeToast in both apps (simulator). Watch for: the leading lightbulb now on shroomsweeper's nudge, the leading alignment vs the old centered text, legibility of `.subheadline`-weight message at large Dynamic Type, the tinted-border float reading correctly in both forest + twilight (the shadow is gone). Apply any `[C: ...]` markers / tweaks (kit changes go on `shroomkit`'s `feature/wave3-nudgetoast` branch; no back-compat shim needed).

- [ ] **Step 6: On approval, commit + merge all three repos**

```bash
# shroomkit
cd ~/dev/games/shroom-games/shroomkit && git checkout main && git merge --ff-only feature/wave3-nudgetoast && git push && git branch -d feature/wave3-nudgetoast
# rootline
cd ~/dev/games/shroom-games/rootline && git add rootline/Views/TutorialView.swift && \
  git commit -m "refactor: use ShroomKit NudgeToast in the tutorial coaching slot

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>" && \
  git checkout main && git merge --ff-only feature/wave3-nudgetoast && git push && git branch -d feature/wave3-nudgetoast
# shroomsweeper (stage only the edited file)
cd ~/dev/games/shroom-games/shroomsweeper && git add shroomsweeper/Views/TutorialView.swift && \
  git commit -m "refactor: use ShroomKit NudgeToast for the board nudge; drop twilight-invisible shadow

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>" && \
  git checkout main && git merge --ff-only feature/wave3-nudgetoast && git push && git branch -d feature/wave3-nudgetoast
```

---

## Component arc B — TutorialBannerCard

### Task 3: `TutorialBannerCard` in ShroomKit

**Files:**
- Create: `shroomkit/Sources/ShroomKit/Components/TutorialBannerCard.swift`

**Interfaces:**
- Consumes: `EyebrowLabel(_ text: String, tint: EyebrowLabel.Tint)` (existing).
- Produces: `public struct TutorialBannerCard<Trailing: View, Footer: View>: View` with
  `public init(eyebrow: String? = nil, title: String, message: String, @ViewBuilder trailing: () -> Trailing = { EmptyView() }, @ViewBuilder footer: () -> Footer = { EmptyView() })`.

- [ ] **Step 1: Branch**

```bash
cd ~/dev/games/shroom-games/shroomkit && git checkout main && git checkout -b feature/wave3-bannercard
```

- [ ] **Step 2: Write `TutorialBannerCard.swift`**

Create `shroomkit/Sources/ShroomKit/Components/TutorialBannerCard.swift`:

```swift
import SwiftUI

/// Tutorial step banner: a pill card wrapping an optional eyebrow + trailing
/// action, a title, a body message, and an optional footer (e.g. a CTA).
/// Flow content is consumer-supplied; the kit owns only the chrome. The
/// header row (eyebrow + trailing) renders only when `eyebrow` is non-nil —
/// the trailing slot rides with the eyebrow.
public struct TutorialBannerCard<Trailing: View, Footer: View>: View {
    private let eyebrow: String?
    private let title: String
    private let message: String
    private let trailing: Trailing
    private let footer: Footer

    @Environment(\.palette) private var palette

    public init(
        eyebrow: String? = nil,
        title: String,
        message: String,
        @ViewBuilder trailing: () -> Trailing = { EmptyView() },
        @ViewBuilder footer: () -> Footer = { EmptyView() }
    ) {
        self.eyebrow = eyebrow
        self.title = title
        self.message = message
        self.trailing = trailing()
        self.footer = footer()
    }

    public var body: some View {
        VStack(alignment: .leading, spacing: 5) {
            if let eyebrow {
                HStack {
                    EyebrowLabel(eyebrow, tint: .accent)
                    Spacer()
                    trailing
                }
            }
            Text(title)
                .font(.system(.title3, design: .rounded).weight(.semibold))
                .foregroundStyle(palette.text)
                .padding(.top, eyebrow == nil ? 0 : 2)
            Text(message)
                .font(.system(.callout, design: .rounded))
                .foregroundStyle(palette.sub)
                .lineSpacing(2)
                .fixedSize(horizontal: false, vertical: true)
            footer
        }
        .padding(.horizontal, Space.md)
        .padding(.vertical, 14)
        .frame(maxWidth: .infinity, alignment: .leading)
        .background(
            RoundedRectangle(cornerRadius: Radius.xxl, style: .continuous)
                .fill(palette.pill)
        )
    }
}
```

- [ ] **Step 3: Run build to verify**

Run: `cd ~/dev/games/shroom-games/shroomkit && swift build 2>&1 | tail -5 && swift test 2>&1 | tail -10`
Expected: build succeeds; existing tests still pass. (No new unit test — the component is pure layout; verification is the downstream app builds + visual review, per the house pattern.)

- [ ] **Step 4: Commit (no merge yet — gated on Task 4 review)**

```bash
cd ~/dev/games/shroom-games/shroomkit
git add Sources/ShroomKit/Components/TutorialBannerCard.swift
git commit -m "feat: add TutorialBannerCard slot-based tutorial banner

Optional eyebrow + trailing (header rides with eyebrow), title, message,
optional footer CTA. Unified type ramp (.title3 / .callout).

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Adopt `TutorialBannerCard` in rootline + shroomsweeper

**Files:**
- Modify: `rootline/rootline/Views/TutorialView.swift` (the `instructionPill` computed property, ~lines 135-154)
- Modify: `shroomsweeper/shroomsweeper/Views/TutorialView.swift` (the `tutorialBanner` computed property, ~lines 68-116)

**Interfaces:**
- Consumes: `TutorialBannerCard` from Task 3.

- [ ] **Step 1: Branch both apps** (keep `shroomkit` on `feature/wave3-bannercard`)

```bash
cd ~/dev/games/shroom-games/rootline && git checkout main && git checkout -b feature/wave3-bannercard
cd ~/dev/games/shroom-games/shroomsweeper && git checkout main && git checkout -b feature/wave3-bannercard
```

- [ ] **Step 2: Replace rootline's `instructionPill`**

In `rootline/rootline/Views/TutorialView.swift`, replace the entire `instructionPill` computed property with:

```swift
    // MARK: Instruction pill

    private var instructionPill: some View {
        TutorialBannerCard(title: flow.lesson.title, message: flow.lesson.instruction)
    }
```

(Slots default empty — rootline's "Skip all / Skip lesson" top bar and post-solve unlock CTA stay where they are. The banner's type grows from `.subheadline`/`.footnote` to the unified `.title3`/`.callout`.)

- [ ] **Step 3: Replace shroomsweeper's `tutorialBanner`**

In `shroomsweeper/shroomsweeper/Views/TutorialView.swift`, replace the entire `tutorialBanner` computed property with:

```swift
    private var tutorialBanner: some View {
        TutorialBannerCard(
            eyebrow: flow.stepLabel,
            title: flow.title,
            message: flow.body
        ) {
            Button(action: onSkip) {
                Text("Skip")
                    .font(.system(.subheadline, design: .rounded).weight(.semibold))
                    .foregroundStyle(palette.sub)
                    .padding(.horizontal, 12)
                    .frame(minHeight: 44)
                    .contentShape(Rectangle())
            }
            .buttonStyle(.plain)
        } footer: {
            if flow.showNextButton {
                Button {
                    withAnimation(.easeOut(duration: 0.2)) { flow.advance() }
                } label: {
                    Text("Got it")
                }
                .buttonStyle(.shroomPrimary)
                .padding(.top, 10)
            }
            if flow.showDoneButton {
                Button("Start foraging", action: onFinish)
                    .buttonStyle(.shroomPrimary)
                    .padding(.top, 10)
            }
        }
    }
```

(All slots lit. The `EyebrowLabel`, title/body type, pill background, and Skip-button 44pt floor are now the kit's; the footer CTA keeps its `.padding(.top, 10)`.)

- [ ] **Step 4: Build both apps**

```bash
cd ~/dev/games/shroom-games/rootline && xcodebuild build -project rootline.xcodeproj -scheme rootline -destination 'generic/platform=iOS Simulator' 2>&1 | tail -5
cd ~/dev/games/shroom-games/shroomsweeper && xcodebuild build -project shroomsweeper.xcodeproj -scheme shroomsweeper -destination 'generic/platform=iOS Simulator' 2>&1 | tail -5
```
Expected: `** BUILD SUCCEEDED **` for both.

- [ ] **Step 5: REVIEW PAUSE**

Stop. Ask Michelle to review TutorialBannerCard in both apps (simulator). Watch for: rootline's banner now larger (`.title3`/`.callout`) — does it crowd the board/coaching slot at large Dynamic Type?; shroomsweeper's banner unchanged in feel (it was already this ramp); the empty-footer trailing gap on rootline's plain card; eyebrow + Skip alignment on shroomsweeper. Apply tweaks (kit changes on `shroomkit`'s `feature/wave3-bannercard` branch).

- [ ] **Step 6: On approval, commit + merge all three repos**

```bash
# shroomkit
cd ~/dev/games/shroom-games/shroomkit && git checkout main && git merge --ff-only feature/wave3-bannercard && git push && git branch -d feature/wave3-bannercard
# rootline
cd ~/dev/games/shroom-games/rootline && git add rootline/Views/TutorialView.swift && \
  git commit -m "refactor: use ShroomKit TutorialBannerCard for the instruction pill

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>" && \
  git checkout main && git merge --ff-only feature/wave3-bannercard && git push && git branch -d feature/wave3-bannercard
# shroomsweeper (stage only the edited file)
cd ~/dev/games/shroom-games/shroomsweeper && git add shroomsweeper/Views/TutorialView.swift && \
  git commit -m "refactor: use ShroomKit TutorialBannerCard for the tutorial banner

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>" && \
  git checkout main && git merge --ff-only feature/wave3-bannercard && git push && git branch -d feature/wave3-bannercard
```

---

### Task 5: Close out Phase 2 in the roadmap

**Files:**
- Modify: `shroomgames-meta/ROADMAP.md` (Wave 3 checkbox, ~line 41)

- [ ] **Step 1: Tick Wave 3**

In `shroomgames-meta/ROADMAP.md`, change the Wave 3 line from `- [ ] **Wave 3 — tutorial flow**: …` to `- [x]` with a one-line completion note (date, both components, net line delta, "merged to main in all 3 repos"), matching the Wave 1/2 entries' style.

- [ ] **Step 2: Commit**

```bash
cd ~/dev/games/shroom-games/shroomgames-meta && git add ROADMAP.md && \
  git commit -m "docs: mark Phase 2 Wave 3 (tutorial scaffold) complete

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>" && git push
```

---

## Self-Review

**Spec coverage:**
- `TutorialBannerCard` (slot-based, unified ramp, header-rides-with-eyebrow) → Task 3 + Task 4. ✓
- `NudgeToast` (one look, tone-tinted border, app-local placement, tone mapping) → Task 1 + Task 2. ✓
- Twilight shadow fix (drop the `.black` shadow, border-only, no new token) → Task 1 (kit, no shadow) + Task 2 Step 3 (deletes the func holding the old shadow). ✓
- A11y (44pt Skip/CTA, tone-prefixed VoiceOver label, EyebrowLabel) → Task 1 (`accessibilityLabel`), Task 4 (Skip keeps `minHeight: 44`, CTA via `.shroomPrimary`). ✓
- Flow state / placement / transitions stay app-side → preserved in Task 2 (rootline anti-jump frame, shroomsweeper overlay) and Task 4 (rootline top bar/unlock strip untouched). ✓
- Verification: `swift build`/`swift test` + generic-destination `xcodebuild` + per-component review pause → every task. ✓

**Placeholder scan:** No TBD/TODO; all code blocks complete; no "handle edge cases". ✓

**Type consistency:** `NudgeTone` (`.guidance`/`.warning`, `iconName`, `accessibilityPrefix`), `NudgeToast(_:tone:)`, `TutorialBannerCard(eyebrow:title:message:trailing:footer:)`, `EyebrowLabel(_:tint:)` consistent across Tasks 1-4. ✓
