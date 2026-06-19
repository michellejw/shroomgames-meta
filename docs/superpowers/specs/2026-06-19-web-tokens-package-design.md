# Web Tokens Package — Design Spec

**Date:** 2026-06-19
**Part of:** Big Rock #2 — design system, **Phase 3 (web foundation)**, scoped to enabling Next.js web games. Triggered by the first web game arriving.
**Depends on:** Phase 1 tokens foundation (the DTCG `tokens.json` + Style Dictionary build in shroomkit). No iOS work involved.

---

## Goal

Make the design tokens consumable by Next.js (Tailwind v4) web games as a real npm package, so a new web game wires in the suite's colors / radii / spacing / type from its first commit instead of hardcoding values. Tokens stay a single source of truth in shroomkit; this adds web outputs and packaging.

## Scope

**In:**
- Extend the existing Style Dictionary build to emit web artifacts (typed JS/TS token object + a Tailwind v4 theme layer) alongside the CSS.
- Package those as `@shroomgames/tokens` at `shroomkit/tokens/dist/`, consumed via a local `file:` path dependency.
- Consolidate the web CSS output into `dist/` (single canonical location) and re-point the marketing site's vendored copy to it.
- A drift check that the JS export stays in sync with the tokens/CSS.

**Out:**
- The web game itself (separate repo, built after this).
- iOS component consolidation (Phase 2).
- Publishing to the npm registry (local `file:` dependency only for now).
- Converting the static marketing site to consume the package as a dependency (it's plain HTML/CSS with no bundler — it stays a vendored CSS copy, just sourced from `dist/`).
- Motion tokens (still not in the token set).

---

## Architecture

```
shroomkit/tokens/tokens.json   (DTCG — single source of truth, unchanged)
        │
        ▼
   Style Dictionary  (shroomkit/tokens/build.mjs, `npm run build:tokens`)
        │
        ├──► Sources/ShroomKit/Theme/Palette.swift          (iOS — unchanged from Phase 1)
        ├──► Sources/ShroomKit/Theme/Tokens.generated.swift (iOS — unchanged)
        │
        └──► shroomkit/tokens/dist/   ← the @shroomgames/tokens package
                 ├── package.json   (hand-authored, committed)
                 ├── tokens.css     (generated — CSS custom properties + [data-theme] blocks)
                 ├── index.js       (generated — typed token object, ESM)
                 ├── index.d.ts     (generated — types)
                 └── theme.css      (generated — Tailwind v4 @theme inline layer)
                        │
                        ├──► web game (file:../shroomkit/tokens/dist)
                        └──► future web games
```

The marketing site (`shroomgames-site`) vendors a copy of `dist/tokens.css` (it can't `npm install`).

---

## The package: `@shroomgames/tokens`

Lives at `shroomkit/tokens/dist/`. `package.json` (hand-authored, committed — the build only writes the artifact files, never `package.json`):

```json
{
  "name": "@shroomgames/tokens",
  "version": "0.1.0",
  "type": "module",
  "exports": {
    ".": { "types": "./index.d.ts", "default": "./index.js" },
    "./tokens.css": "./tokens.css",
    "./theme.css": "./theme.css"
  },
  "files": ["index.js", "index.d.ts", "tokens.css", "theme.css"]
}
```

Consumed in the game:
- `import "@shroomgames/tokens/tokens.css"` (global, in `app/layout.tsx`) — defines the `--shroom-*` / `--radius-*` / `--space-*` vars and the `[data-theme]` blocks.
- `@import "@shroomgames/tokens/theme.css"` in the game's Tailwind entry — maps Tailwind namespaces onto those vars.
- `import { tokens } from "@shroomgames/tokens"` — raw typed values for game logic / canvas rendering.

## Generated outputs

### `tokens.css`
Identical content to today's Phase 1 output (CSS custom properties, `:root, [data-theme="forest"]` + `[data-theme="twilight"]` blocks, scales). Emitted into `dist/` instead of `tokens/build/`.

### `index.js` + `index.d.ts`
A typed, `as const` nested object mirroring the token tree, **including both themes' raw hex** (a puzzle game needs real per-theme values for canvas drawing, where CSS vars don't reach):

```ts
export const tokens = {
  color: {
    forest:   { appBg: "#F3EFE4", accent: "#6E8B4E", /* …all 23… */ numberColors: ["#5C8C57", /* …8… */] },
    twilight: { /* …23… */ numberColors: [/* …8… */] },
  },
  radius: { sm: 10, md: 12, lg: 14, xl: 16, "2xl": 18, pill: 22 },
  space:  { "2xs": 4, xs: 8, sm: 12, md: 16, lg: 24, xl: 32 },
  font: {
    weight:   { regular: 400, medium: 500, semibold: 600, bold: 700 },
    tracking: { normal: 0, eyebrow: 1.3, wide: 2 },
    design:   { rounded: "rounded", mono: "monospaced" },
  },
} as const
```

`numberColors` here is the 1–8 color list (no `.clear` sentinel — that's a SwiftUI concern). The `warn` alias is not duplicated; consumers use `mushroomCap` (web already does).

### `theme.css` (Tailwind v4)
A `@theme inline` layer mapping Tailwind's expected namespaces (`--color-*`, `--radius-*`, `--spacing-*`) onto the `--shroom-*` / `--radius-*` / `--space-*` vars from `tokens.css`. `inline` is what keeps the utilities resolving through the live CSS vars, so `data-theme` switching flows through Tailwind utilities (`bg-shroom-accent` recolors on theme change). **The exact Tailwind v4 `@theme inline` syntax must be verified against current Tailwind docs during implementation, not assumed.**

## Theming guidance for the game (integration notes — not package code)

- Set `data-theme="forest"` (default) on `<html>`; a toggle flips to `twilight` and persists to `localStorage` (same model as the marketing site's `theme-toggle.js`).
- Avoid an SSR flash of the wrong theme with a tiny inline `<head>` script that sets `data-theme` from `localStorage` before first paint (standard Next.js pattern; `next-themes` also solves this).

## Testing

- **JS↔token parity:** a Node check asserting representative `tokens.*` values match `tokens.json` (e.g. `tokens.color.forest.accent === "#6E8B4E"`, `tokens.radius.md === 12`), so the JS output can't silently drift from the source.
- **Determinism / no-drift:** extend the existing `check:tokens` script to also rebuild and `git diff --exit-code` the `dist/` artifacts.
- **Package resolves:** a check that `dist/package.json` `exports` point at files that exist after a build.

## Build/structure notes

- The build harness stays at `shroomkit/package.json` (`build:tokens`); the consumable `dist/package.json` is separate and hand-authored.
- Generated files in `dist/` carry the `// Generated …` / `/* Generated … */` header and are committed (like Phase 1's outputs).
- Removing the old `tokens/build/` CSS output: update the marketing site's vendored `generated.css` to be copied from `dist/tokens.css`.

## Open questions (decide during planning)

- Exact Tailwind v4 theme-layer mechanism (`@theme inline` vs `@theme` + a mapping file) — confirm against current Tailwind v4 docs.
- JS export tooling: a Style Dictionary `javascript/esm` + `typescript` format, or a small custom format for the `as const` shape. Default to whatever produces clean `as const` + `.d.ts`; settle when scaffolding.
- Whether `theme.css` is generated by Style Dictionary or hand-authored (it's mostly a static mapping). Lean toward generated so token additions propagate.
