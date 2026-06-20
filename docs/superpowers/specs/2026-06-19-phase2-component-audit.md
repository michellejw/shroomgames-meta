# Phase 2 — iOS Component Consolidation: Audit & Extraction Inventory

**Date:** 2026-06-19
**Part of:** Big Rock #2 — design system, Phase 2 (iOS component consolidation).
**Status:** Audit complete (autonomous). Extraction design + decisions pending Michelle's review.

This is the audit deliverable: which UI patterns are genuinely shared across **rootline** and **shroomsweeper** and should move into ShroomKit. Built from a full read of both apps' `Views/`.

## Already shared (no action)
- `LoadingView`, `WelcomeScaffold` — both apps already consume the ShroomKit versions (shroomsweeper's local `LoadingView`/`WelcomeView` are thin wrappers, not duplicates).
- `Palette`, `Appearance`, `ThemeMode`, `Radius`/`Space`/`FontTracking` tokens — in ShroomKit.

## Two cross-cutting findings (fold into every extraction)
1. **Token adoption gap.** BOTH apps hardcode radii (`12/14/16/18/22`), spacing, and `tracking(1.3)` as literals instead of using the `Radius`/`Space`/`FontTracking` tokens that already exist. Phase 1 deferred adoption; Phase 2 is when extracted components start consuming the tokens.
2. **Accessibility gaps** (high-value given Michelle runs large Dynamic Type): missing `.accessibilityAddTraits(.isSelected)` on toggles/selection rows; icon-only buttons lack `.accessibilityLabel`; a few CTAs land ~42pt (under the 44pt floor); `.caption2` eyebrow may be too small at large type; a tutorial shadow is hardcoded `.black` (near-invisible in twilight). Extraction is the one place to fix each of these once.

## Extraction candidates — present in BOTH apps (prioritized)

### Wave 1 — Primitives (small, self-contained, highest reuse; adopting them ripples cleanup through both apps)
| Candidate | In rootline | In shroomsweeper | Notes |
|---|---|---|---|
| **PillIconButton** (44pt icon in pill/circle) | 7+ (PlayView ×4, headers) | HomeView, WelcomeView, GameView | Highest repetition in the suite. Bake in 44pt + a REQUIRED `accessibilityLabel`. |
| **Primary / Secondary button** (accent CTA; pill-fill + tier-bg/border ghost variants) | HomeView, TutorialView, cards | HomeView, OptionsSheet, TutorialView, ResultBar, WinEntryCard | Radii vary (14/16/18) → unify. |
| **EyebrowLabel** (caps + tracking 1.3 + sub) | HomeView, PlayView, StatsView, SettingsSheet | HomeView, TutorialView | `FontTracking.eyebrow` token exists but is unused; consider `.caption2`→`.caption` for large type. |
| **StatPill** (icon + monospaced readout) | PlayView (timer, hints) | GameView | Explicitly listed in ShroomKit CLAUDE.md. |

### Wave 2 — Composites
| Candidate | In rootline | In shroomsweeper | Notes |
|---|---|---|---|
| **ScreenHeader** (back button + title + trailing slot) | DifficultyView, StatsView, PlayView | GameView title row | `@ViewBuilder trailing` for optional actions (e.g. Stats "Clear"). |
| **SegmentedToggle** (2-up pill toggle) | PlayView + TutorialView (verbatim dup) | GameView + TutorialView (verbatim dup) | Duplicated *within* each app too. Add `.isSelected` trait. |
| **SelectionCard** (radio difficulty/mode row) | DifficultyView `tierRow` | OptionsSheet `difficultyRow` | Title + subtitle + radio dot + selected chrome. |
| **ResultCard** (badge slot + title/subtitle + secondary/primary button pair) | WinCard, RevealedCard | ResultBar, WinEntryCard | ShroomKit CLAUDE.md "ResultBar". Mascot via `@ViewBuilder` slot like WelcomeScaffold. |
| **Settings chrome** (`SettingsSection`, `SelectionChip`, `SettingsRow`) | SettingsSheet | OptionsSheet (tabbed) | Section header + selection chip + icon/label/chevron row. |

### Wave 3 — Tutorial flow (most app-coupled; do last, validate carefully)
| Candidate | Notes |
|---|---|
| **TutorialBannerCard** + **NudgeToast** | Both apps have a `TutorialView` + `TutorialFlow` with a step banner and nudge toasts. High value but the most coupled to per-app flow state — extract the chrome, keep flow content app-supplied. Fix the twilight shadow here. |

## Extraction discipline (per ShroomKit CLAUDE.md)
For each component: build it in ShroomKit → swap BOTH apps to use it → if the swap feels awkward, the API isn't done. No back-compat shims (two consumers, update together). Review-pause per component before moving to the next.

## Out of scope
- App-specific mascots/icons (`MyceliumIcon`, `MushroomIcon`) — consumer-supplied via slots.
- Game logic (board/cell/edge mechanics, solving, scoring).
- A full literal-sweep of every hardcoded value app-wide — only what extracted components cover (a broader sweep can be its own pass).
