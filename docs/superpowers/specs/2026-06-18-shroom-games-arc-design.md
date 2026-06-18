# Shroom Games — Full Arc Design

**Date:** 2026-06-18
**Scope:** Suite-wide plan across all three big rocks, plus the agreed implementation detail for each.

---

## Overview

Three big rocks in order. Each one is a prerequisite for the next in terms of UX maturity and architectural readiness.

```
1. Calm leaderboard rework    (~days)    — ships UX improvement, road-tests calm stats pattern
2. Design tokens system       (~weeks)   — single source of truth before more surfaces lock in
3. Solver + generator + daily (~months)  — transforms Rootline into an evergreen daily game
```

Shroomsweeper is mostly a passenger: it may inherit the calm stats pattern after #1 if it feels right, and it automatically benefits from tokens once ShroomKit consumes them.

---

## Big Rock #1: Calm leaderboard rework

**Scope:** Rootline only. Six files, one deleted.

### Changes

| File | Action |
|------|--------|
| `Views/WinEntrySheet.swift` | Delete — 3-letter initials entry sheet gone entirely |
| `Storage/ScoreStore.swift` | Rewrite — store best time + completion count per tier; no ranked top-5 |
| `Views/BestTimesView.swift` | Rewrite + rename to `StatsView.swift` — one line per tier: *"Your fastest Sprout: 1:23 · 23 cleared"* |
| `Views/WinCard.swift` | Update — whisper time only if it beats the stored best (*"0:42 — your fastest yet"*); otherwise silent on time |
| `Views/PlayView.swift` | Update — remove entry-sheet trigger on win; call new ScoreStore API |
| `Views/HomeView.swift` | Update — link to StatsView instead of old leaderboard entry point |

### After shipping
Sit with the pattern. If it feels right, it becomes a candidate for ShroomKit and Shroomsweeper retrofits to match.

---

## Big Rock #2: Design tokens system

**Scope:** ShroomKit + all web surfaces (marketing site, web game, future web games).

### Decisions

| Question | Decision | Rationale |
|----------|----------|-----------|
| Token spec | DTCG format (design-tokens.org W3C community group) | Suite growing to 5+ games; spec compliance unlocks Figma plugins and future tooling without custom adapters |
| Generator | Style Dictionary | Standard tool for JSON→Swift+CSS; avoids a custom build script to maintain |
| Where tokens live | `tokens/` directory inside ShroomKit repo | Co-located with the Swift extensions it generates; one less repo |
| Figma sync | Manual for now | Plugin adds complexity; not in critical path at this scale |

### Architecture

```
tokens/tokens.json  (DTCG format — single source of truth)
         │
         ▼
   Style Dictionary
   (build step in ShroomKit)
         │
         ├──► Swift extensions ──────────► ShroomKit
         │    (Tokens.swift)                   │
         │                                     ├──► rootline
         │                                     ├──► shroomsweeper
         │                                     └──► future iOS games
         │
         ├──► tokens.css ────────────────► shroomgames-site
         │    (CSS custom properties)
         │
         ├──► tokens.css ────────────────► web-game
         │
         └──► tokens.css ────────────────► future web games
```

### Sequence

1. Audit hard-coded values in `Palette.swift` (and any spacing/radius values in use) — this becomes the token inventory.
2. Write `tokens/tokens.json` in DTCG format covering palette, typography, spacing, radii.
3. Configure Style Dictionary with three output targets: Swift extensions, CSS for marketing site, CSS for web game.
4. Wire build step into ShroomKit so tokens regenerate when the JSON changes.
5. Update `shroomgames-site/` to consume CSS output.
6. Add web game output target to SD config when that project starts.

---

## Big Rock #3: Solver + generator + daily/archive

**Scope:** Rootline. Four phases in strict order.

### Phase 1 — Solver (Swift CLI, no UI)
Standalone CLI under `scripts/`. Takes a Slitherlink grid with clues, determines whether a unique solution exists.

- Constraint propagation first (fast, eliminates most candidates)
- Backtracking search second (for hard cases)
- Uniqueness check (confirm exactly one solution — not zero, not two+)

Developed and tested in isolation before any generator work begins.

### Phase 2 — Generator (Swift CLI)
Builds on the solver.

- Generate random grids
- Derive solution via existing `Engine.swift`
- Hide clues strategically
- Run solver to confirm uniqueness
- Grade difficulty by counting techniques the solver needed
- Emit JSON bundle of validated puzzles per tier

Output: a bundle that ships with the app as a bundled resource.

### Phase 3 — App-side: daily + archive
- Load JSON bundle as bundled resource
- Map date → puzzle deterministically (date hash mod count)
- Home screen: prominent "Today's grove" entry point
- Archive view: scrollable grid of past days, each with cleared / not / streak status
- Replaces current Sprout #1/#2/#3 cycling

### Phase 4 — Migrate persistence
Swap grove-index-keyed progress for date-keyed puzzle IDs. Decide at implementation time whether to migrate existing progress best-effort or start fresh.

---

## What's not changing

- No sharing solutions
- No competitive leaderboard (replaced by calm rework)
- No hints changes
- No accounts / cloud sync
