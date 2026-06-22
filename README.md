# Shroom Games

A small suite of calm, offline iOS puzzle games and the shared SwiftUI design system that backs them. Solo project by Michelle Weirathmueller.

This repo holds the suite-wide planning material — the [`ROADMAP`](./ROADMAP.md), implementation plans, design specs. The apps and the design system live in sibling repos.

## The suite

| Repo | What it is |
| --- | --- |
| [`mycogrid/`](../mycogrid/) · [github](https://github.com/michellejw/mycogrid) | Mycogrid loop puzzle — game 2 (Xcode project/target still named `rootline` internally) |
| [`shroomsweeper/`](../shroomsweeper/) · [github](https://github.com/michellejw/shroomsweeper) | Cozy Minesweeper — game 1 |
| [`shroomkit/`](../shroomkit/) · [github](https://github.com/michellejw/shroomkit) | Shared SwiftUI design system (palette, theme, scaffolds). Local Swift Package consumed by both apps. |
| [`shroomgames-site/`](../shroomgames-site/) | Marketing site at [shroomgames.app](https://shroomgames.app) |
| [`shroomgames-meta/`](.) | (this repo) Suite-wide ROADMAP, planning docs, design notes |

## What's in this repo

- [`ROADMAP.md`](./ROADMAP.md) — the standing plan across the suite
- [`docs/superpowers/plans/`](./docs/superpowers/plans/) — implementation plans
- [`docs/superpowers/specs/`](./docs/superpowers/specs/) — design specs

## Package wiring (for reference)

Each app references ShroomKit as a **local-path** Swift Package (`../shroomkit`), not a URL. URL-based was attempted but Xcode's SPM resolver was flaky — punted. Locally this just works.
