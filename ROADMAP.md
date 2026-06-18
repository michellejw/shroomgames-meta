# Rootline + Shroom Games roadmap

A standing plan for where Rootline (and the broader Shroom Games suite) is headed. Things settled here have been agreed already; revisit only when something genuinely changes.

## Progress tracker

### Big Rock #1 — Calm leaderboard rework
- [ ] Delete `WinEntrySheet.swift`
- [ ] Rewrite `ScoreStore.swift` (best time + completion count per tier; no ranked top-5)
- [ ] Rewrite + rename `BestTimesView.swift` → `StatsView.swift`
- [ ] Update `WinCard.swift` (always show time; add "Your fastest yet" whisper only if beaten)
- [ ] Update `PlayView.swift` (remove entry-sheet trigger; call new ScoreStore API)
- [ ] Update `HomeView.swift` (link to StatsView)
- [ ] Add "X puzzles cleared" total to StatsView
- [ ] Sit with the pattern — decide if it goes into ShroomKit for Shroomsweeper

### Big Rock #2 — Design system (phased)

Reframed from "design tokens" — the real goal is a Shroom Games design system across iOS + web. Three layers: **values** (tokens), **components** (reusable views; ShroomKit is already half of this), **cross-platform parity** (same values/patterns on web). Tokens come first because every component on both platforms consumes them.

**Phase 1 — Tokens foundation** (the values layer)
- [ ] Audit current values: colors in `Palette.swift` + scattered radii/spacing in app views + type conventions (font design/weight/tracking layered on Dynamic Type)
- [ ] Write `tokens/tokens.json` in DTCG format (colors, spacing, radii, typography conventions)
- [ ] Configure Style Dictionary: Swift extensions (ShroomKit) + CSS custom properties (web)
- [ ] Wire build step into ShroomKit (tokens regenerate when JSON changes); regenerate `Palette.swift` from tokens
- [ ] Update `shroomgames-site/` to consume CSS output
- [ ] Xcode smoke-check: both apps still build against regenerated ShroomKit

**Phase 2 — iOS component consolidation** (later)
- [ ] Audit patterns duplicated across rootline/shroomsweeper (button styles, stat pill, result bar, eyebrow label, etc.)
- [ ] Pull shared ones into ShroomKit; make all components consume tokens

**Phase 3 — Web foundation** (later; web components gated on the web game existing)
- [ ] Marketing site fully on CSS tokens
- [ ] Web components when the web game repo exists

**Cross-cutting patterns** (woven in as needed)
- [ ] Tutorial scaffold parity
- [ ] Calendar component — co-design with Big Rock #3's archive view (don't build speculatively)

### Big Rock #3 — Solver + generator + daily/archive
**Phase 1 — Solver**
- [ ] Implement constraint propagation
- [ ] Implement backtracking search
- [ ] Implement uniqueness check
- [ ] CLI runnable and tested in isolation

**Phase 2 — Generator**
- [ ] Random grid generation
- [ ] Clue-hiding strategy
- [ ] Difficulty grading (technique count)
- [ ] Emits validated JSON bundle per tier

**Phase 3 — App-side**
- [ ] Load JSON bundle as bundled resource
- [ ] Date → puzzle mapping (deterministic hash)
- [ ] "Today's grove" entry point on Home
- [ ] Archive view (scrollable grid, cleared/not/streak status)
- [ ] Remove Sprout #1/#2/#3 cycling

**Phase 4 — Migrate persistence**
- [ ] Decide: migrate existing progress best-effort or start fresh
- [ ] Implement date-keyed puzzle ID storage

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

### 3. Slitherlink solver + generator + daily/archive UI

Replace the hand-curated puzzle pool with an offline-generated bundle, surfaced as NYT-style daily puzzles plus a browseable archive.

**Architecture**:
- An **offline Swift CLI generator** under `scripts/` produces a JSON bundle of validated puzzles per tier:
  - Random region generation
  - Solution derivation (already in `Engine.swift`)
  - Clue-hiding strategy
  - **Slitherlink solver** (constraint propagation + uniqueness check) — the hardest piece
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

- **Rootline** ([github.com/michellejw/rootline](https://github.com/michellejw/rootline)) — v1 feature-complete per the original design plan. Plus: app icon, persistence, puzzle editor + add-puzzle script, tutorial overhaul, leaderboard (the soon-to-be-reworked arcade version), theme cycle in-game, full Dynamic Type pass.
- **Shroomsweeper** ([github.com/michellejw/shroomsweeper](https://github.com/michellejw/shroomsweeper)) — on ShroomKit, persistence, ThemeMode, in-game theme cycle.
- **ShroomKit** ([github.com/michellejw/shroomkit](https://github.com/michellejw/shroomkit)) — Palette, Appearance, ThemeMode, LoadingView, WelcomeScaffold. Two consumers. Local-path package dependency (URL-based was attempted but Xcode's SPM resolver was flaky — punted).

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
