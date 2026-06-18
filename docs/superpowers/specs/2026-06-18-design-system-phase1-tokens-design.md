# Design System — Phase 1: Tokens Foundation (Design Spec)

**Date:** 2026-06-18
**Part of:** Big Rock #2 — Shroom Games design system (phased). This is Phase 1 of 3 (see `ROADMAP.md`).
**Depends on:** nothing. Everything else in the design system consumes this.

---

## Goal

A single source of truth for the suite's design *values* — colors, radii, spacing, and typography conventions — authored once and emitted to both platforms: Swift constants for ShroomKit (iOS) and CSS custom properties for web. Replaces hand-coded values, starting with the colors ShroomKit already centralizes.

## Scope

**In:**
- A DTCG-format token source (`shroomkit/tokens/tokens.json`).
- A Style Dictionary build that emits Swift (ShroomKit) + CSS (web).
- **Migrate colors now:** regenerate `Palette.swift` from the color tokens.
- **Define + emit (but don't yet adopt)** radii, spacing, and typography tokens — available for use, not yet wired into every call site.
- CSS custom properties consumed by `shroomgames-site/`.

**Out (later phases):**
- Migrating app views (rootline/shroomsweeper) to consume radii/spacing/type tokens — that's **Phase 2** (incremental, snapping values as each view is touched).
- Web *components*; the marketing site only consumes CSS vars here.
- Figma sync automation (manual for now).
- Motion/animation tokens (not in this cut).

**Why this keeps Phase 1 low-risk:** the only surface that changes behavior now is color (regenerated from tokens, values identical). Radii/spacing/type are additive — emitted and ready, adopted gradually in Phase 2 — so the rationalized scales cause no big-bang visual shift.

---

## Token categories

### 1. Colors — faithful 1:1

`Sources/ShroomKit/Theme/Palette.swift` is the **authoritative source of the hex values**; the implementer transcribes them exactly into `tokens.json` (no value changes).

- Two themes: `forest` (light), `twilight` (dark).
- 23 semantic tokens per theme: `appBg, boardBg, text, sub, pill, accent, accentText, tileCovered, tileCoveredHi, tileCoveredEdge, tileRevealed, tileRevealedEdge, mushroomCap, mushroomStem, mushroomSpot, explodeBg, markerPost, markerSign, markerInk, emptyMark, tierBorder, tierBg, tierSelBg`.
- `numberColors`: an ordered 9-slot array per theme; slot 0 is `.clear` (a sentinel, not a real color — keep it as the literal `.clear` in the generated Swift, not a token).
- `warn` is a **derived alias** of `mushroomCap` — express as a token reference (DTCG alias), not a duplicated value.

### 2. Radii — rationalized to 6 named steps

| token | value (pt/px) |
|---|---|
| `radius.sm` | 10 |
| `radius.md` | 12 |
| `radius.lg` | 14 |
| `radius.xl` | 16 |
| `radius.2xl` | 18 |
| `radius.pill` | 22 |

(Audit snap mapping, for Phase 2 adoption reference: 9→sm, 13→md, 20→2xl, 26→pill. Values 0/1/2 are hairline/square one-offs — left as literals, not tokenized.)

### 3. Spacing — tight 4pt scale

| token | value |
|---|---|
| `space.2xs` | 4 |
| `space.xs` | 8 |
| `space.sm` | 12 |
| `space.md` | 16 |
| `space.lg` | 24 |
| `space.xl` | 32 |

(Phase 2 adoption snaps the heavy odd values: 6→xs, 10→xs/sm by context, 14→sm/md, 18→md, 22→lg.)

### 4. Typography — primitives + recipes (size stays semantic)

**Do not tokenize font size.** iOS sizing is semantic Dynamic Type (`.system(.title2, …)`), which the user depends on; web uses its own type scale. Tokenize only the house-style primitives applied *on top of* size:

Primitive tokens:
- `font.design`: `rounded` (default), `mono` (digits/timers).
- `font.weight`: `regular` (400), `medium` (500), `semibold` (600, default), `bold` (700).
- `font.tracking`: `normal` (0), `eyebrow` (1.3), `wide` (2).

Recipe text styles (documented; emitted as helpers, not raw tokens):
- **default** — rounded design + semibold weight + a semantic role chosen at the call site.
- **eyebrow** — caps + `tracking.eyebrow` + `.caption2` role + `sub` color (the existing tracked-caps label).
- **mono-digit** — `monospacedDigit` for times/counts.

The Swift output ships these as view-modifier helpers; CSS ships font-family/weight/letter-spacing utility classes. Size is never baked in.

---

## Pipeline

```
shroomkit/tokens/tokens.json   (DTCG, hand-authored — the source of truth)
        │
        ▼
   Style Dictionary  (Node; config + npm script in the shroomkit repo)
        │
        ├──► Sources/ShroomKit/Theme/Tokens.generated.swift   (radii, spacing, type primitives)
        ├──► Sources/ShroomKit/Theme/Palette.swift             (regenerated from color tokens)
        └──► tokens.css                                        (CSS custom properties → shroomgames-site/)
```

**Decisions (settled in arc brainstorming):**
- Format: **DTCG** (design-tokens.org W3C community group).
- Generator: **Style Dictionary** (multi-platform output from one source).
- Home: **`tokens/` directory inside the ShroomKit repo**, co-located with generated Swift.
- ShroomKit is a SwiftPM package → `swift build`/`swift test` run headlessly. Only the final both-apps build-check needs Xcode.

**Generated-file conventions:**
- Generated Swift carries a "// Generated by Style Dictionary — do not edit" header.
- `Color(hex:)` extension stays hand-authored (it's logic, not a value); the generated palette uses it.
- The generator is run via an explicit npm script (e.g. `npm run build:tokens`); it is **not** wired into `swift build` (keep SwiftPM free of a Node dependency). Regeneration is a deliberate step, committed alongside the token change.

---

## Output shapes

### Swift (ShroomKit)
- `Palette.swift` — same public API as today (`Palette` struct, `forest`/`twilight` statics, `palette(for:)`, `warn`). Only the literal values are now generated. **Public surface must not change** — both apps consume it unchanged.
- `Tokens.generated.swift` — `enum Radius { static let sm: CGFloat = 10; … }`, `enum Space { … }`, and the type primitives. Namespaced so call sites read `Radius.md`, `Space.sm`.

### CSS (web)
- `tokens.css` — `:root { --color-forest-accent: #6E8B4E; … }` plus a `twilight` variant (via `prefers-color-scheme` or a `.twilight` class — implementer picks the simpler one for the current site), and `--radius-md`, `--space-sm`, `--font-weight-semibold`, etc.

---

## Testing

- **ShroomKit (`swift test`, headless):** a test asserting the regenerated `Palette` values match the known-good hex (guards against a bad regeneration). Assert `warn == mushroomCap`. Assert `numberColors` has 9 entries with slot 0 `.clear`.
- **Token build:** a check that `npm run build:tokens` produces output with no Style Dictionary errors and the generated files are non-empty.
- **Generated-vs-committed drift:** optional CI-style check that re-running the generator yields no diff (committed output is up to date).
- **Xcode smoke-check (delegated):** after regeneration, both `rootline` and `shroomsweeper` build and look unchanged. This is the one GUI-bound step.

---

## Open questions (decide during planning)

- CSS theming mechanism: `prefers-color-scheme` media query vs `.twilight` class — pick based on how `shroomgames-site/` currently handles dark mode.
- Whether `Tokens.generated.swift` radii/spacing emit as `CGFloat` (SwiftUI-friendly) — yes, default to `CGFloat`.
- Exact npm/Style Dictionary version pinning — settle when scaffolding the build.
