# LithicTTD

Design and technical documentation for a Stone Age settlement game: OpenTTD's visual style,
gameplay inspired by Banished and Transport Tycoon Deluxe, with a Civilization-style
technology tree.

## Documents

| Document | Contents |
|---|---|
| [01 — OpenTTD feasibility](docs/01-feasibility.md) | Can OpenTTD serve as the base? What you get free, what you must build, what fights you. Licence analysis, subsystem-by-subsystem LOC accounting, comparison against alternative bases. |
| [02 — Engine rework plan](docs/02-engine-rework-plan.md) | 15 phases from fork hygiene to modding, with per-phase scope, file-level touchpoints, exit gates, and a risk register. |
| [03 — Game logic guidelines](docs/03-game-logic-guidelines.md) | Conventions for writing the simulation: determinism, commands, amortisation, performance budgets, save/load, tech gating, testing, anti-patterns. |
| [04 — OpenRCT2 feasibility](docs/04-openrct2-feasibility.md) | The same investigation applied to OpenRCT2. Rejected as a base — but four architectural patterns worth importing, now folded into doc 02. |

## Summary

**Recommended base: an OpenTTD fork, with two conditions.**

1. **OpenTTD is GPL v2 (not "or later").** Your game's source becomes GPL v2. Selling it is
   fine; closed-source distribution, console ports, and GPL-incompatible middleware are not.
   This is a business decision that must be settled before any code is written.
2. **You are forking a game, not adopting an engine.** Of OpenTTD's 403,957 lines, roughly
   130k are keepable, ~167k get deleted, and ~61k get rewritten. There is no plugin seam that
   produces a village simulation.

**What the fork buys you** — 12–18 months of work you don't do: a battle-tested isometric
renderer with correct occlusion sorting on sloped terrain (the hardest problem in this art
style, and the reason the game will *look* right), 6 zoom levels, SIMD blitters and an OpenGL
backend, 8bpp *and* full RGBA 32bpp, terrain generation and terraforming, a chunked versioned
save system, a complete GUI toolkit, a localisation pipeline, and an amortised tile-simulation
loop that is already the right shape for forest regrowth and crop growth.

**What it does not buy you** — the actual game: agent pathfinding (OpenTTD's YAPF is
track-based and inapplicable), individual villagers with needs and lifecycles, the job and
hauling system, buildings that consume labour, a non-monetary resource economy, and the
technology tree. Budget years, not months.

## Why not OpenRCT2

It was the strongest alternative and got the same full investigation
([doc 04](docs/04-openrct2-feasibility.md)). Rejected on two constraints a fork cannot engineer
away: **it requires purchased RollerCoaster Tycoon 2 game files to run** (~29,357 sprite indices
baked to the original `g1.dat` layout, versus OpenTTD's freely-shippable base set), and it is
**hard-locked to a 256-colour palette** with no 32bpp path anywhere in the codebase.

The reason it looked attractive — that its "peeps" are free-roaming agents with needs and
pathfinding — **does not survive reading the code.** Guests walk only on placed footpath tiles;
the pathfinder is an explicitly-not-A\* depth-first search returning one direction per step; and
of the `Guest` struct's ~40 fields, three transfer to a villager. **Neither engine gives you
agent pathfinding** — that is work you do regardless of base.

Its tile-element model, JSON+PNG asset objects, replay/desync tooling, and entity tweener *are*
better than OpenTTD's, and all four have been imported into the rework plan.

**Before committing:** settle the licence question, then build the 3-week vertical spike in
[Phase 0](docs/02-engine-rework-plan.md#phase-0--fork-hygiene--vertical-spike) — ten villagers
walking to trees, chopping, and hauling wood on an otherwise unmodified fork. That spike answers
the three questions that decide the project.

---

*Assessed against OpenTTD [`be4f099`](https://github.com/OpenTTD/OpenTTD) and OpenRCT2
[`4e576a0`](https://github.com/OpenRCT2/OpenRCT2), both 2026-08-10, on Ubuntu 24.04 / GCC 13.3.
OpenTTD: full clean build 524 s on 4 cores, 94 test cases / 2,145 assertions passing. OpenRCT2:
CMake configure failed on missing `libcurl` in the same environment.*
