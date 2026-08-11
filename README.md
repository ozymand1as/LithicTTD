# LithicTTD

Design and technical documentation for a Stone Age settlement game: OpenTTD's visual style,
gameplay inspired by Banished and Transport Tycoon Deluxe, with a Civilization-style
technology tree.

## Documents

| Document | Contents |
|---|---|
| [01 — Feasibility assessment](docs/01-feasibility.md) | Can OpenTTD serve as the base? What you get free, what you must build, what fights you. Licence analysis, subsystem-by-subsystem LOC accounting, and a comparison against alternative bases. |
| [02 — Engine rework plan](docs/02-engine-rework-plan.md) | 15 phases from fork hygiene to modding, with per-phase scope, file-level touchpoints, exit gates, and a risk register. |
| [03 — Game logic guidelines](docs/03-game-logic-guidelines.md) | Conventions for writing the simulation: determinism, commands, amortisation, performance budgets, save/load, tech gating, testing, anti-patterns. |

## Summary

**Feasible, with two conditions.**

1. **OpenTTD is GPL v2 (not "or later").** Your game's source becomes GPL v2. Selling it is
   fine; closed-source distribution, console ports, and GPL-incompatible middleware are not.
   This is a business decision that must be settled before any code is written.
2. **You are forking a game, not adopting an engine.** Of OpenTTD's 403,957 lines, roughly
   130k are keepable, ~167k get deleted, and ~61k get rewritten. There is no plugin seam that
   produces a village simulation.

**What the fork buys you** — 12–18 months of work you don't do: a battle-tested isometric
renderer with correct occlusion sorting on sloped terrain (the hardest problem in this art
style, and the reason the game will *look* right), 6 zoom levels, SIMD blitters and an OpenGL
backend, terrain generation and terraforming, a chunked versioned save system, a complete GUI
toolkit, a localisation pipeline, and an amortised tile-simulation loop that is already the
right shape for forest regrowth and crop growth.

**What it does not buy you** — the actual game: agent pathfinding (OpenTTD's YAPF is
track-based and inapplicable), individual villagers with needs and lifecycles, the job and
hauling system, buildings that consume labour, a non-monetary resource economy, and the
technology tree. Budget years, not months.

**Before committing:** settle the licence question, spend one day evaluating OpenRCT2 (whose
"peeps" are already free-roaming agents with needs and pathfinding, at the cost of GPL v3 and
a park-shaped tile model), then build the 3-week vertical spike in
[Phase 0](docs/02-engine-rework-plan.md#phase-0--fork-hygiene--vertical-spike) — ten villagers
walking to trees, chopping, and hauling wood on an otherwise unmodified fork. That spike
answers the three questions that decide the project.

---

*Assessed against OpenTTD [`be4f099`](https://github.com/OpenTTD/OpenTTD) (2026-08-10).
Build and unit tests verified on Ubuntu 24.04 / GCC 13.3: full clean build 524 s on 4 cores,
94 test cases / 2,145 assertions passing.*
