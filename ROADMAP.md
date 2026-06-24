# Mycogrid + Shroom Games roadmap

A standing plan for where Mycogrid (and the broader Shroom Games suite) is headed. Things settled here have been agreed already; revisit only when something genuinely changes.

> **Naming (settled 2026-06-22):** the loop-puzzle game is **Mycogrid** — repo `mycogrid` (`github.com/michellejw/mycogrid`), local `~/dev/games/shroom-games/mycogrid`. It was formerly called "rootline" (and never "Slitherlink"). The Xcode project/target and its source folder are *still named `rootline` internally* — a deliberate residual, not yet renamed. Dated specs/plans and session logs predating this note say "rootline"; read those as Mycogrid.

## Progress tracker

### Big Rock #1 — Calm leaderboard rework
- [x] Delete `WinEntrySheet.swift`
- [x] Rewrite `ScoreStore.swift` (best time + completion count per tier; no ranked top-5)
- [x] Rewrite + rename `BestTimesView.swift` → `StatsView.swift`
- [x] Update `WinCard.swift` (always show time; add "Your fastest yet" whisper only if beaten)
- [x] Update `PlayView.swift` (remove entry-sheet trigger; call new ScoreStore API)
- [x] Update `HomeView.swift` (link to StatsView)
- [x] Add "X puzzles cleared" total to StatsView
- [ ] Sit with the pattern — decide if it goes into ShroomKit for Shroomsweeper

**Bonus rework shipped alongside** (not originally on the roadmap, surfaced while playtesting):
- [x] Hints simplified to one tap = one move (was 3-tap escalation); unlimited with "N hints used" counter
- [x] "Show solution" reveal button — view-only, doesn't record stats; gated by confirm alert
- [x] Eye button relocated next to back to prevent accidental taps adjacent to hint
- [x] Back button now confirms ("Leave puzzle?") when the player has made moves

### Big Rock #2 — Design system (phased)

Reframed from "design tokens" — the real goal is a Shroom Games design system across iOS + web. Three layers: **values** (tokens), **components** (reusable views; ShroomKit is already half of this), **cross-platform parity** (same values/patterns on web). Tokens come first because every component on both platforms consumes them.

**Phase 1 — Tokens foundation** (the values layer)
- [x] Audit current values: colors in `Palette.swift` + scattered radii/spacing in app views + type conventions (font design/weight/tracking layered on Dynamic Type)
- [x] Write `tokens/tokens.json` in DTCG format (colors, spacing, radii, typography conventions)
- [x] Configure Style Dictionary: Swift extensions (ShroomKit) + CSS custom properties (web)
- [x] Wire build step into ShroomKit (`npm run build:tokens`); `Palette.swift` regenerated from tokens, `Tokens.generated.swift` for radii/spacing/type
- [x] Update `shroomgames-site/` to consume CSS output (zero visual change — all 65 color values verified)
- [x] Xcode smoke-check: rootline (Mycogrid) TestFlight archive built clean against regenerated ShroomKit; shroomsweeper shares the same frozen `Palette` API

*Phase 1 complete (2026-06-19). Merged to `main` in shroomkit + shroomgames-site. Phase-2 follow-ups (in shroomkit `.git/sdd` ledger): adopt radii/spacing tokens in app views; derive `PALETTE_ORDER` from token metadata.*

**Phase 2 — iOS component consolidation** (in progress — done in waves; audit + extraction inventory in `docs/superpowers/`)
- [x] Audit patterns duplicated across rootline/shroomsweeper (~13 shared patterns found; 3 waves planned)
- [x] **Wave 1 — primitives** (2026-06-20): `EyebrowLabel`, `PillIconButton` (+ `ThemeMode.iconName`), `StatPill`, `ShroomButtonStyle` (primary/secondary/outline). Both apps swapped; a11y fixed on extraction (required icon labels, .caption eyebrow, 44pt floor); CTA radius unified + pressed-state. Net −400 lines of duplicated inline styling. Merged to main in all 3 repos.
- [x] **Wave 2 — composites** (2026-06-20): `ScreenHeader`, `SelectionCard`, `SegmentedToggle`, `ResultCard`, settings chrome (`SettingsSection`/`SelectionChip`/`SettingsRow`). Both apps swapped where applicable; `.isSelected` traits added (last a11y gap from the audit); the 4-file-duplicated mode toggle + three end-of-game cards collapsed. Net −370 lines. Merged to main in all 3 repos.
- [x] **Wave 3 — tutorial flow** (2026-06-21): `TutorialBannerCard` (slot-based: optional eyebrow + trailing + footer CTA; unified `.title3`/`.callout` ramp) and `NudgeToast` (+`NudgeTone` guidance/warning, tone-tinted border). Both apps swapped (rootline keeps its anti-jump coaching frame; shroomsweeper keeps its board-overlay slide-in). Killed the twilight-invisible hardcoded `.black` nudge shadow — the tinted border carries float in both themes. Tone-prefixed VoiceOver labels added. Merged to main in all 3 repos. **Phase 2 complete.**
  - Bonus parity (2026-06-21): gave shroomsweeper a "Leave game?" / "Start over?" confirm before discarding an in-progress board (Home + Reset), matching rootline's calm-rework "Leave puzzle?" guard.
- [ ] Decide if the calm-stats pattern (Big Rock #1) gets a shared component too
- [ ] **Deferred from Wave 2** — game-screen headers (rootline PlayView, shroomsweeper GameView) left bespoke; `ScreenHeader` is slot-based so they can adopt later. Decide whether they should structurally match or look identical.
- [ ] **Cross-app theme control consistency.** Quick toggle is now a 2-state light/dark flip (3-state cycle had a dead click). rootline keeps "System" in its Settings picker; shroomsweeper is light/dark-only (no settings picker). Unify: give shroomsweeper a settings theme control too (a shared `ThemePicker`?), and make the 2-state flip read live appearance so there's no first-tap no-op from System.

**Phase 3 — Web foundation** (kicked off 2026-06-19 — triggered by the first web game)
- [x] `@shroomgames/tokens` consumable npm package at `shroomkit/tokens/dist/` — typed JS export (both themes), `tokens.css`, Tailwind v4 `theme.css` (`@theme inline`); consumed via local `file:` dep. Spec/plan in `docs/superpowers/`.
- [x] Marketing site consumes the generated CSS tokens (vendored copy, re-pointed to `dist/`)
- [x] Scaffold the web game and wire it to `@shroomgames/tokens` (import `tokens.css` + `theme.css`, theme via `data-theme`) — **Nonagarden** (Next.js 16 + React 19 + Tailwind v4), consumes the tokens via `file:` dep; Fredoka via `next/font`; `data-theme` toggle w/ anti-flash script. Core-play slice shipped 2026-06-20.
- [ ] **Add web typography to the token package.** `@shroomgames/tokens` ships weights + tracking but no web *font family* or *type scale*; its `font.design` values (`rounded`/`monospaced`) are SwiftUI `Font.Design`, not valid CSS. Each web game currently re-invents this locally (Nonagarden supplies Fredoka via `next/font` + a small local type scale in `globals.css`). Decide on a web font + scale and emit them from the token source so games don't each duplicate it.
- [ ] **Resolve radius scale drift for game surfaces.** The nonogram design wants tile `9` / board `20` / sheet `28`px, none of which exist in the token scale (`10/12/14/16/18/22`). Nonagarden uses local px literals for tiles for now. Decide: add game/grid radii to `tokens.json`, or accept per-game local literals for board-specific shapes.
- [ ] Web components (shared React patterns) once the game reveals what's reusable

*Web follow-ups surfaced 2026-06-19 while scaffolding Nonagarden (the first web game) + wiring the token package — see the two checked items above.*

**Cross-cutting patterns** (woven in as needed)
- [ ] Tutorial scaffold parity
- [ ] Calendar component — co-design with Big Rock #3's archive view (don't build speculatively)

### Big Rock #3 — Solver + generator + daily/archive
**Phase 1 — Solver**
- [ ] Implement constraint propagation
- [ ] Implement backtracking search
- [ ] Implement uniqueness check
- [ ] CLI runnable and tested in isolation

**Phase 2 — Generator** *(complete 2026-06-22 — merged to rootline main `b95c357`; spec+plan in `docs/superpowers/{specs,plans}/2026-06-21-rock3-generator*`)*
- [x] Random grid generation — cell accretion + no-hole flood-fill guard (`RegionGenerator`)
- [x] Clue-hiding strategy — greedy hide to the pure-logic cap (`ClueHider`); density emerges
- [~] Difficulty grading (technique count) — `meta.rulesFired` (clue/dot tallies) emitted per puzzle as a signal, but tier stays = grid size (not solver-graded). Full grading deferred; revisit only if real boards feel mis-tiered.
- [x] Emits validated JSON bundle per tier — `mycogrid-generate` CLI; pure-logic gate (`unique && guesses==0`); byte-reproducible (seeded SplitMix64 + deterministic solver); content-derived stable ids; self-audited via `mycogrid-validate`

Fast-follow (non-blocking, from final review): `generateOne` does a redundant final `solve` purely to capture `rulesFired` for meta — `ClueHider` could return its final trace instead (perf on big sparse tiers).

**Phase 3 — App-side** *(complete 2026-06-24 — merged to mycogrid main; spec+plan in `docs/superpowers/{specs,plans}/2026-06-22-rock3-phase3-app*`; handoff in `docs/superpowers/handoffs/2026-06-23-rock3-phase3-part2-handoff.md`)*
- [x] Load JSON bundle as bundled resource — `PuzzleBundle` decode bridge + `puzzles.json` (700 entries, integrity audited via `mycogrid-validate bundle`)
- [x] Date → puzzle mapping (deterministic, append-only occurrence-index — NOT hash mod) — `DailyService`
- [x] "Today's grove" entry point on Home
- [x] Archive view (calendar layout, weekday columns, cleared/streak status)
- [x] Remove Sprout #1/#2/#3 cycling — Difficulty picker retired, "Next puzzle" treadmill removed

**Phase 4 — Migrate persistence** *(complete 2026-06-24 — folded into Phase 3 merge; pre-TestFlight so no install base to migrate)*
- [x] Decide: migrate existing progress best-effort or start fresh — fresh start, no migration code (no install base)
- [x] Implement date-keyed puzzle ID storage — id-keyed `CompletionStore` is the single source of truth; `ProgressStore` re-keyed to `{puzzleID, playedDate}` (`_v2`); `ScoreStore` retired

## Guiding principles

- **Calm and meditative** is the north star for Rootline's UX. No arcade/competitive framing.
- **Works offline, no accounts, no ads.** (Same as the original design plan.)
- **Cross-app consistency** matters — anything visual or interaction-pattern-y should be sharable across the suite (iOS apps + future web).
- **The owner (Michelle) is up for ambitious work** to learn how to build solid apps.

## Three big rocks (in order)

### 1. Calm-leaderboard rework

Replace the current arcade-style leaderboard (3-letter initials, ranked top 5, "New record!" entry sheet) with a quieter pattern:

- Track best time per tier silently
- Show as a single line per tier in a "Stats" view: *"Your fastest Sprout: 1:23"* + optional completion count (*"23 sprouts cleared"*)
- The win card whispers "**0:42 — your fastest yet**" if you beat your own best, otherwise stays quiet on time
- No 3-letter entry sheet at all
- No ranks

This is the smallest of the three. Doing it first lets us sit with what "calm stats" feels like before committing to a shared pattern.

**Affected files**: `Storage/ScoreStore.swift`, `Views/BestTimesView.swift`, `Views/WinEntrySheet.swift` (delete), `Views/WinCard.swift`, `Views/PlayView.swift`, `Views/HomeView.swift`.

If this pattern feels right, it becomes a candidate for ShroomKit, and Shroomsweeper retro-fits to it. Shroomsweeper currently has the arcade leaderboard; doesn't have to change unless we decide it should match.

### 2. Design system (phased) across iOS + web

A Shroom Games design system, not just tokens. Three layers — **values** (tokens), **components** (reusable views; ShroomKit already holds the iOS half), **cross-platform parity** (same values/patterns on web). Built in phases (see the Phase 1–3 checklist in the Progress tracker); tokens come first because every component on both platforms consumes them.

**Phase 1 — Tokens foundation.** Single source of truth for colors, spacing, radii, and typography *conventions*. The last one matters: typography here is font design (`.rounded`) + weight + tracking layered on top of Dynamic Type's sizing — house-style choices, not pixel sizes (Michelle runs large Dynamic Type and depends on semantic sizing). A `tokens/` source plus a build pipeline that emits:

- Swift extensions for ShroomKit — replaces the hand-coded values in `Palette.swift` / future `Spacing.swift` / etc.
- CSS custom properties for the marketing site (`shroomgames.app`) and any future web games
- (Manual but tracked) Figma library updates

**Why now**: before more apps and more web surfaces ship and lock in duplicates of the values.

**Settled decisions** (from arc brainstorming):
- Tokens spec: **DTCG format** (design-tokens.org W3C community group) — suite is growing to 5+ games incl. a web app; spec compliance unlocks Figma/tooling without custom adapters.
- Generator: **Style Dictionary** — standard JSON→Swift+CSS tool; multi-target config covers each surface.
- Tokens live in a **`tokens/` directory inside the ShroomKit repo** — co-located with the Swift extensions it generates.
- Figma sync: **manual** for now.
- ShroomKit is a SwiftPM package → `swift build` / `swift test` run headlessly; only the final both-apps build-check needs Xcode.

**Later phases**: iOS component consolidation (pull patterns duplicated across rootline/shroomsweeper into ShroomKit, consuming tokens), then web foundation (site on CSS tokens; web components once the web-game repo exists). The reusable **calendar** is co-designed with #3's archive view, not built speculatively.

### 3. Mycogrid solver + generator + daily/archive UI

Replace the hand-curated puzzle pool with an offline-generated bundle, surfaced as NYT-style daily puzzles plus a browseable archive.

**Architecture**:
- An **offline Swift CLI generator** under `scripts/` produces a JSON bundle of validated puzzles per tier:
  - Random region generation
  - Solution derivation (already in `Engine.swift`)
  - Clue-hiding strategy
  - **Mycogrid solver** (constraint propagation + uniqueness check) — the hardest piece
  - Difficulty grading (count of techniques the solver needs)
- The output JSON ships as an app **bundled resource**
- App loads the bundle, maps date → puzzle deterministically (e.g., date hash mod count)
- **Daily** view: "Today's grove" prominent on Home
- **Archive** view: scrollable grid of all past days, each with a status (cleared / not / streak)
  - This *is* the "completion calendar" — a quiet record of which puzzles you've cleared. It only becomes meaningful here, once puzzles have stable date-based identities (the fixed pool in #1 just cycles, so there's nothing durable to put on a calendar). The calm-stats screen from #1 holds the total count until then.
- Replaces the current Sprout #1 / #2 / #3 cycling

**Why deepest**: solver is real work, takes time to get right, but unlocks endless content forever. The reason the original design plan punted on it.

**Order within #3**:
1. Solver (CLI, validates uniqueness, no UI)
2. Generator (CLI, emits JSON bundle)
3. App-side: load bundle, daily, archive
4. Migrate progress/persistence from grove-index to date-keyed puzzle id

## Status of where we are right now (as of last commit)

All three apps build clean, public on GitHub, MIT-licensed.

- **Mycogrid** ([github.com/michellejw/mycogrid](https://github.com/michellejw/mycogrid)) — v1 feature-complete. Calm per-tier stats + one-tap hint + show-solution reveal shipped on top. Big Rock #3 solver + generator (Phases 1–2) merged. Next likely step: TestFlight beta. (Xcode project still named `rootline` internally.)
- **Shroomsweeper** ([github.com/michellejw/shroomsweeper](https://github.com/michellejw/shroomsweeper)) — on ShroomKit, persistence, ThemeMode, in-game theme cycle. Still on the arcade leaderboard pattern; will revisit once we've sat with Mycogrid's calm-stats screen.
- **ShroomKit** ([github.com/michellejw/shroomkit](https://github.com/michellejw/shroomkit)) — Palette, Appearance, ThemeMode, LoadingView, WelcomeScaffold. Two consumers. Local-path package dependency (URL-based was attempted but Xcode's SPM resolver was flaky — punted). Design-system phase 1 (tokens) in flight.
- **shroomgames-meta** ([github.com/michellejw/shroomgames-meta](https://github.com/michellejw/shroomgames-meta)) — this repo. Suite-wide ROADMAP + planning docs + design specs, versioned alongside (not inside) the app repos.

## Repo housekeeping

The Shroom Games repos are scattered awkwardly across the filesystem and the locations are misleading. Cleaning this up is overdue.

**Current state:**
- `~/dev/archive/APPS/puzzle-game/rootline/` — active, but lives under "archive"
- `~/dev/archive/APPS/puzzle-game/shroomsweeper/` — active, but lives under "archive"
- `~/dev/games/shroomkit/` — closer to right, but still flat with whatever else lands in `~/dev/games/`
- `~/dev/sites/shroomsweeper-site/` — the suite marketing site (`shroomgames.app`), misleadingly named after Shroomsweeper

**Target state:**
```
~/dev/games/
    shroom-games/        # the suite lives here
        rootline/
        shroomsweeper/
        shroomkit/
        shroomgames-site/    # currently ~/dev/sites/shroomsweeper-site/
        shroomgames-meta/    # suite-wide ROADMAP, planning docs, design notes
    {future non-shroom games}/
```
The Shroom Games suite is its own thing; `~/dev/games/` stays general so future non-Shroom games can live alongside it.

**Things to handle in the move:**
- Update each app project's local Swift Package reference path (currently `../../../../games/shroomkit` from the buried location; becomes `../shroomkit` after the move — siblings under `shroom-games/`)
- Update any hardcoded paths in scripts (e.g., `scripts/add-puzzle.swift` only uses repo-relative paths, so probably fine)
- Update CLAUDE.md "Active Projects" listings (machine-local context file)
- Verify Xcode opens cleanly after the move (it remembers absolute paths in some cases)
- Marketing site directory rename: `shroomsweeper-site/` → `shroomgames-site/` while we're at it

Not urgent but worth doing before more apps land — every new app added to the current scattered layout makes this more painful.

## What's not on the roadmap (intentionally)

- **Sharing solutions** — not interested
- **Per-puzzle initials/competitive leaderboard** — replaced by the calm rework
- **Accounts / cloud sync** — design plan says no, sticking with that
