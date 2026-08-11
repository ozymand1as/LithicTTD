# Feasibility: OpenTTD as the base for a Lithic-era settlement game

**Scope of this assessment.** Can OpenTTD (as of `be4f099`, 2026-08-10) serve as the base
for a Banished-inspired, TTD-inspired settlement game about Stone Age communities, with a
Civilization-style technology tree?

**Verdict: yes, technically feasible, and defensible — but only if you accept two things.**

1. You are not "using OpenTTD as an engine." You are **forking a finished game** and
   harvesting roughly a third of it as an engine, deleting roughly 40%, and rewriting the
   rest. There is no plugin seam that gets you a village sim.
2. **OpenTTD is GPL v2 (not v2-or-later).** Your game's source becomes GPL v2. See
   [Licensing](#1-licensing-the-decision-that-comes-before-the-code) — this is the one
   finding that can kill the plan outright, and it is a business decision, not an
   engineering one.

Everything below is measured against the actual source tree, not from memory.

---

## Contents

- [1. Licensing](#1-licensing-the-decision-that-comes-before-the-code)
- [2. What the codebase actually is](#2-what-the-codebase-actually-is)
- [3. What you get for free — the real value](#3-what-you-get-for-free--the-real-value)
- [4. What you must build from nothing](#4-what-you-must-build-from-nothing)
- [5. What actively fights you](#5-what-actively-fights-you)
- [6. Gameplay-to-engine mapping](#6-gameplay-to-engine-mapping)
- [7. Alternatives, honestly compared](#7-alternatives-honestly-compared)
- [8. Recommendation](#8-recommendation)

---

## 1. Licensing: the decision that comes before the code

`COPYING.md` is GNU GPL **version 2**, and every source file carries a header naming
*"version 2"* with no "or later" clause:

```
OpenTTD is free software; you can redistribute it and/or modify it under the terms of
the GNU General Public License as published by the Free Software Foundation, version 2.
```

Consequences, stated plainly:

| | |
|---|---|
| **Your game code** | Must be GPL v2. Any file you link into the binary is a derivative work. |
| **Selling it** | Allowed. GPL restricts distribution terms, not payment. Steam/itch sales are fine. |
| **Source disclosure** | Mandatory for anyone who receives the binary. You cannot ship closed-source. |
| **Your art & audio** | *Not* forced to be GPL — they are separate works, merely data the program reads. Keep them proprietary if you wish. Sprites baked into a linked `.grf`-producing build step are a grey area; keep assets as loose data files to stay clearly on the safe side. |
| **Third-party middleware** | GPL v2 is incompatible with a lot of commercial middleware (notably some analytics/anti-cheat/DRM SDKs, and Apache-2.0 code, which is GPLv2-incompatible). Check every dependency. |
| **Console ports** | Effectively blocked. Platform NDAs conflict with GPL source disclosure. |
| **Trademarks** | You may not imply endorsement or reuse the OpenTTD name/logo. Rename the project and binary early. |
| **Original TTD graphics** | Proprietary. Cannot ship. OpenGFX (GPL v2) can be, but it looks like OpenTTD — you want your own art anyway. |

**Practical read:** a GPL v2 indie release is a well-trodden path (OpenRCT2, Widelands,
Endless Sky derivatives). But if your plan involves a closed-source commercial release, a
console port, or proprietary middleware, **stop here** and use a permissively licensed base
(see [§7](#7-alternatives-honestly-compared)).

---

## 2. What the codebase actually is

403,957 lines of C++ across ~1,300 files (excluding vendored third-party). Modern C++20:
`std::unique_ptr`, `EnumBitSet`, strong typedefs, `<=>`. Well-documented, consistent
`CODINGSTYLE.md`, Doxygen throughout. Genuinely good code — this matters, because you will
be reading a *lot* of it.

I classified every file by fate under a Lithic reskin:

| Fate | LOC | Share | Contents |
|---|---:|---:|---|
| **Keep as-is** — core/util | 26,319 | 6.5% | `core/` (pool, kdtree, bitmath, random), `misc/`, `timer/` |
| **Keep as-is** — render/platform | 26,630 | 6.6% | `video/`, `blitter/`, `sound/`, `gfx.cpp`, `spritecache.cpp`, palette |
| **Keep, heavy edit** — viewport/landscape | 21,123 | 5.2% | `viewport.cpp`, `landscape.cpp`, `tile_*`, `terraform`, `genworld`, `tgp` |
| **Keep as-is** — GUI framework | 20,709 | 5.1% | `window.cpp`, `widget.cpp`, `widgets/`, dropdowns, sortlists |
| **Keep, port** — saveload | 20,999 | 5.2% | Chunked versioned serialiser |
| **Keep** — settings, strings, command fw | 15,358 | 3.8% | `settings*`, `strings*`, `lang/`, `command.cpp` |
| **Delete** — rail | 22,056 | 5.5% | Track, signals, PBS, depots, trains |
| **Delete** — road/tram | 14,940 | 3.7% | |
| **Delete** — ship/water infra | 4,457 | 1.1% | Docks, canals, locks |
| **Delete** — air | 6,847 | 1.7% | |
| **Delete** — vehicles | 41,961 | 10.4% | Vehicle/engine/orders/groups/autoreplace/timetables/bridges/tunnels |
| **Delete** — NewGRF | 25,724 | 6.4% | 87 files of extension VM for TTD content |
| **Delete** — network | 23,585 | 5.8% | Lockstep multiplayer, coordinator, admin port |
| **Delete** — script/AI | 27,857 | 6.9% | Squirrel AI + GameScript API (158 API files) |
| **Rework** — economy/station/town/industry | 60,962 | 15.1% | Cargo, stations, link graph, money, companies, towns, industries |
| **Misc / infra** | 44,430 | 11.0% | `openttd.cpp`, console, fileio, screenshots, framerate, misc GUIs |

Roughly: **~130k LOC kept, ~167k deleted, ~61k rewritten, ~44k triaged.**

The build works out of the box. `cmake -B build -DOPTION_DEDICATED=ON` configured cleanly on
stock Ubuntu 24.04 / GCC 13.3 with only `zlib`, `liblzma`, `libpng`, `freetype`, `fontconfig`
present — no vcpkg, no vendored-dependency hunt. **Full clean build: 524 s on 4 cores**
(~9 minutes), producing a 20 MB binary; the bundled unit-test suite passes clean
(**94 test cases, 2,145 assertions**). Incremental single-file rebuilds are seconds.
Iteration speed is a non-issue. (Caveat: `openttd_lib` is one large target, so touching a
widely-included header rebuilds the world — budget for header hygiene when adding your own.)

---

## 3. What you get for free — the real value

### 3.1 The isometric renderer (the actual reason to do this)

This is the crown jewel and the single best argument for forking rather than starting fresh.

`ViewportAddLandscape()` (`src/viewport.cpp:1211`) walks visible tiles, each tile emits
sprites through a per-type callback, and then everything — terrain, buildings, trees, moving
actors — is resolved by a **topological sort of 3D bounding boxes**:

```c
// src/viewport_sprite_sorter.h:16
struct ParentSpriteToDraw {
	int32_t xmin, ymin, zmin, x;   // 16B block, xmm-loadable
	int32_t xmax, ymax, zmax, y;
	SpriteID image; PaletteID pal; const SubSprite *sub;
	int32_t left, top;             // reference point for child sprites
	int32_t first_child; uint32_t order;
};
```

`ViewportSortParentSprites()` (`viewport.cpp:1600`) does the dependency ordering, with an
SSE4.1 path selected at runtime (`viewport.cpp:3619`). Getting correct occlusion for
arbitrary-sized objects on sloped isometric terrain is *the* hard problem of this art style,
and it is solved, debugged, and shipped here. Rewriting this correctly is months of work and
a long tail of visual glitches.

You also inherit:

- **Sloped terrain rendering with foundations.** `Slope`, `GetFoundation`, half-tile slopes,
  `GetSlopePixelZ()` (`landscape.cpp:313`) giving exact ground height at any sub-tile
  position — precisely what walking actors need on hills.
- **6 zoom levels** (`ZoomLevel::In4x` … `Out8x`, `zoom_type.h:20`) with per-zoom sprite
  variants.
- **8bpp and 32bpp blitters** including SSE2/SSSE3/SSE4 optimised paths, plus a 40bpp
  animated-palette blitter and a full **OpenGL backend** (`video/opengl.cpp`,
  `video/sdl2_opengl_v.cpp`). Full RGBA sprites are supported — you are not stuck with a
  256-colour palette.
- **Viewport spatial hash** for moving actors (`ViewportAddVehicles`, `vehicle.cpp:1149`) —
  a bucketed hash so only on-screen actors are considered. Reusable verbatim.
- **Dirty-rect invalidation** (`MarkTileDirtyByTile`), smooth scrolling, follow-object,
  minimap, screenshot of the whole map.

### 3.2 Tile-map infrastructure

```c
// src/map_func.h:32
struct TileBase {      // 8 bytes
	uint8_t type;      // type (4..7), bridges (2..3), rainforest/desert (0..1)
	uint8_t height;    // height of northern corner
	uint16_t m2; uint8_t m1, m3, m4, m5;
};
struct TileExtended {  // 4 bytes
	uint8_t m6, m7; uint16_t m8;
};
```

12 bytes/tile in two parallel arrays, `TileIndex` a strong 32-bit typedef, map size
64×64 → 4096×4096 (`MIN/MAX_MAP_SIZE_BITS` = 6/12). `TILE_SIZE = 16` world units per tile,
`TILE_PIXELS = 32`, `MAX_TILE_HEIGHT = 255`. For a Banished-scale map (256×256–512×512)
you're using a fraction of the capacity.

Comes with: terraforming with cost/validation (`terraform_cmd.cpp`), procedural terrain gen
(`tgp.cpp`, 1,087 lines of Perlin-ish generation), heightmap import (`heightmap.cpp`),
tile-area/iteration helpers, `Map::Iterate()`, and a k-d tree (`core/kdtree.hpp`, with
`FindNearest`/`FindContained`) already used for towns/stations — ideal for "nearest berry
bush / nearest stockpile" queries.

### 3.3 The tile-type dispatch table — your main extension seam

This is the architectural feature that makes the reskin tractable. Tile behaviour is a
hand-rolled vtable indexed by tile type:

```c
// src/landscape.cpp:69
const EnumIndexArray<const TileTypeProcs *, TileType, TileType::MaxSize> _tile_type_procs = {
	&_tile_type_clear_procs, &_tile_type_rail_procs, &_tile_type_road_procs, ...
};
```

`TileTypeProcs` (`tile_cmd.h`) holds `draw_tile_proc`, `get_slope_pixel_z_proc`,
`clear_tile_proc`, `tile_loop_proc`, `get_tile_desc_proc`, `terraform_tile_proc`,
`animate_tile_proc`, and others. **There are 4 bits of tile type (16 slots), 11 used.**

You replace the 11 transport types with your own set — `Clear`, `Water`, `Forest`, `Rock`,
`Building`, `Field`, `Path`, `Stockpile`, `Resource` — implement one `TileTypeProcs` struct
per type, and the entire viewport/landscape/terraform/tile-info stack keeps working
untouched. This is the difference between "a rewrite" and "a fork."

### 3.4 The amortised tile loop — exactly Banished's simulation pattern

```c
// src/landscape.cpp:806
void RunTileLoop()
{
	/* pseudorandom tile order via maximal-length Galois LFSR */
	uint count = 1 << (Map::LogX() + Map::LogY() - TILE_UPDATE_FREQUENCY_LOG);
	TileIndex tile = _cur_tileloop_tile;
	while (count--) {
		_tile_type_procs[GetTileType(tile)]->tile_loop_proc(tile);
		tile = TileIndex{(tile.base() >> 1) ^ (-(int32_t)(tile.base() & 1) & feedback)};
	}
	_cur_tileloop_tile = tile;
}
```

Every tile gets visited once per `TILE_UPDATE_FREQUENCY` ticks, in a deterministic
pseudorandom order, at constant cost per tick. This is the right way to run soil fertility,
forest regrowth, crop growth, decay and weathering over a large map — and it's already
written, deterministic, and save-safe.

The existing tile loops are directly instructive templates. `TileLoop_Trees`
(`tree_cmd.cpp:836`) is a forest growth/spread simulation: growth stages, density,
neighbour seeding, climate rules, self-limiting spread. `TileLoop_Clear`
(`clear_cmd.cpp:284`) grows grass through density levels and ages farm fields. Rename
"trees" to "woodland" and you have your forestry model's skeleton for free.

### 3.5 Determinism and the command pattern

Every state mutation goes through a registered command with a `DoCommandFlag::QueryCost`
"test" pass followed by an `Execute` pass (`command_type.h:391`), returning `CommandCost`.
Combined with an explicit seeded `Randomizer` (never `rand()`) and integer-only simulation
maths, this gives you:

- Free undo-safety on player actions (test before commit).
- **Replay-based regression testing** — record a command stream, replay it, assert identical
  final state. This is the single most valuable testing tool for a simulation game, and it
  falls out of the architecture.
- A future path to multiplayer if you ever want it.

### 3.6 Pool allocator and serialisation

`core/pool_type.hpp` gives you slot-reusing, index-stable, iterable, save-aware containers.
`Town`, `Industry`, `Station`, `Vehicle`, `Object`, `Goal`, `Sign`, `StoryPage` are all
pools. Your `Villager`, `Building`, `Family`, `Herd`, `WorkOrder` pools drop straight in and
get iteration, index-stability across saves, and `Pool::CleanPool` lifecycle for free.

`saveload/` is a mature chunked, versioned serialiser with per-struct declarative field
tables and a backward-compat layer (`saveload/compat/`). Adapting it is mechanical.

### 3.7 GUI framework

`window.cpp` + `widget.cpp` + `widgets/` (~20k LOC) is a complete retained-mode windowing
toolkit: declarative nested widget trees, auto-layout, resizing, sort/filter lists,
dropdowns, scrollbars, tooltips, hotkeys, string input, and a mature tooltip/query
infrastructure. It is idiosyncratic but battle-tested and — critically — it *looks like
OpenTTD*, which is part of what you're asking for.

Plus a full localisation pipeline (`strgen/`, `lang/`, plural forms, gendered strings,
parameter reordering) that is a genuine pain to build yourself.

---

## 4. What you must build from nothing

This is the honest cost side. None of the following exists in any reusable form.

### 4.1 Agent pathfinding — the biggest gap

Banished is fundamentally about individual people walking around. **OpenTTD has no
general-purpose pathfinder.** YAPF (`pathfinder/yapf/`, ~2,400 LOC) is entirely built
around `Trackdir` — discrete track directions on rails/roads with signal reservation,
`FollowTrack`, and per-transport cost models. It is structurally inapplicable to a villager
crossing open grass.

What *is* useful as precedent: disaster vehicles (`disaster_vehicle.cpp`, 1,029 lines) and
effect vehicles (`effectvehicle.cpp`) move freely across the map in world coordinates,
ignoring tracks entirely — UFOs, zeppelins, submarines. They prove the movement/rendering
substrate supports free-roaming actors; they just do it with hardcoded scripted paths.

You will write: a grid A* with jump-point search or a hierarchical/flow-field pathfinder,
sub-tile steering, slope-aware traversal cost, path caching and invalidation on terrain
change, and crowd handling. Budget **6–10 weeks** for something good, and expect it to be
your #1 performance hotspot forever.

Note the movement resolution you inherit: `TILE_SIZE = 16` world units per tile, so
sub-tile position has 1/16-tile granularity. Fine for villagers; if you want smoother
motion you can widen the position fields (they're already `int32_t x_pos, y_pos, z_pos`).

### 4.2 Individual agents with needs, jobs, and lifecycle

`Vehicle` (`vehicle_base.h:198`) is the closest thing to an agent, and it is the wrong
shape: ~120 fields covering consists (`next`/`previous`/`first`/`last` chains), shared
orders, cargo payment, refit lists, breakdowns, reliability, service dates, NewGRF caches,
liveries, group membership, unit numbers. Harvest the *movement and rendering* fields
(`x_pos`/`y_pos`/`z_pos`, `direction`, `coord`, the viewport hash pointers,
`sprite_cache`) and the tick dispatch; discard the rest. A `Villager` should be a lean
struct, because you'll have thousands.

Then build: age/health/hunger/warmth/happiness, family formation, birth and death,
education, skill levels, inventory carried, and per-agent state machines. All new.

### 4.3 The job/labour system

Banished's core loop is: buildings post work requirements → idle villagers claim tasks →
walk → gather/haul/build → deposit. OpenTTD has nothing analogous. Its closest analogue is
industry production, which is a pure per-tick rate multiplier on a tile with no labour at
all:

```c
// src/industry.h — production is just rate * level, no workers involved
struct ProducedCargo { CargoType cargo; uint16_t waiting; uint8_t rate; ... };
uint8_t prod_level;
```

You will build a work-order queue, task claiming and reservation, hauling logistics,
priority/starvation handling, and construction-site progress. This is the actual game.
Budget generously — **8–12 weeks** and expect continuous tuning.

### 4.4 The technology tree

Nothing exists. There is no research, no unlock gating, no prerequisite graph. What you can
lean on: the `Engine` availability model has an `introduction_date`/`end_of_life_date` +
`IsAvailable()` gating pattern (mirrored in `ObjectSpec`, `newgrf_object.h:57`) that shows
how "buildable thing becomes available over time" threads through the build UI. Replace
"date" with "tech unlocked" and the plumbing shape transfers; the research system itself is
yours.

Since your tech tree is being designed separately, the integration contract matters more
than the implementation — see [Game Logic Guidelines §Tech gating](03-game-logic-guidelines.md#7-technology-gating).

### 4.5 Storage, hauling, and a non-monetary economy

OpenTTD's economy is money-and-transport-distance shaped: `CargoPacket` tracks source and
distance travelled for payment; `Station`'s `GoodsEntry` models waiting cargo and ratings;
`linkgraph/` computes cargo flow across a transport network. None of that models "a
granary holding 400 units of grain that villagers walk to."

The `Money`/`CommandCost` coupling is deep and touches everything — every command returns a
cost, and `Company` holds the balance. You are not deleting money so much as neutralising
it. See the rework plan for the recommended approach (keep `CommandCost` as an
error-carrying vessel, zero the money dimension).

### 4.6 Miscellaneous new systems

Seasons and weather with gameplay effect (OpenTTD has snow line and climate zones as
cosmetic/growth modifiers only); food spoilage; wildlife and hunting; disease; fire;
per-villager clothing/tools; construction sites; a settlement-scale UI (population,
resource dashboard, villager inspector, research screen).

---

## 5. What actively fights you

### 5.1 The graphics pipeline is GRF-bound

**This is the most under-appreciated obstacle.** There is exactly one `SpriteLoader`
implementation for real art:

```
src/spriteloader/grf.hpp:16   class SpriteLoaderGrf : public SpriteLoader
src/spriteloader/makeindexed.cpp   (a 32bpp→8bpp adapter, not an alternative source)
```

`ReadSprite()` (`spritecache.cpp:454`) instantiates `SpriteLoaderGrf` directly. All
graphics arrive as sprites inside GRF containers, indexed by numeric `SpriteID` with a
1,801-line hardcoded table of ~1,192 `SPR_*` constants (`table/sprites.h`), and the game
**cannot start without a base graphics set** — the repo ships no `bin/baseset/` art, and
`LoadSpriteTables()` (`gfxinit.cpp:171`) requires `BaseGraphics::GetUsedSet()`. There is a
whole `bootstrap_gui.cpp` dedicated to nagging the user into downloading one.

Two ways out:

| Option | Effort | Verdict |
|---|---|---|
| Use the NML/`grfcodec` toolchain to build your own base set from PNGs (what OpenGFX does) | Low up front | Ties your art pipeline to a niche 1990s container format and its tooling forever. Fine for a prototype. |
| **Write a `SpriteLoaderPNG`**: PNG atlases + a JSON/INI manifest declaring offsets, bounds, zoom variants; register sprites by symbolic name instead of numeric ID | ~1–2 weeks | **Recommended.** `SpriteLoader` is a clean virtual interface (`spriteloader.hpp:100`), and a named-sprite registry removes the hardcoded ID table. Do this in Phase 2, before you commission art. |

The second option is one of the highest-leverage decisions in the whole project: it
determines whether your artists work in a modern pipeline or fight `nforenum` for three
years.

### 5.2 Tile storage is fully packed

The `m1`–`m8` bytes are exhaustively bit-allocated across the existing tile types
(documented in `docs/landscape.html`). A settlement sim wants per-tile soil fertility,
resource quantity, growth stage, walkability, reservation-by-agent, path wear, snow depth —
that does not fit in 12 bytes alongside everything else.

Since you're forking, just widen it. Deleting rail/road/air frees the majority of the bit
budget, and going to 16 or 20 bytes/tile costs 1.3 MB on a 512×512 map. **But**: do the
widening *deliberately and early*, with a documented bit-allocation table, or you'll ship
the same undocumented bit-packing chaos that makes OpenTTD's tile layer hard to modify.

### 5.3 Everything assumes companies

`Owner`/`CompanyID` is threaded through tiles (`m1`), commands, and every ownership check.
A single-settlement game has one implicit owner. Don't try to excise `Owner` — collapse it
to a single always-present company and leave the plumbing. Fighting this is a waste of
weeks.

### 5.4 Global mutable state

`_settings_game`, `_cur_tileloop_tile`, `_vehicle_viewport_hash`, `_tile_type_procs`,
`_game_mode` and friends are globals. This is idiomatic for the codebase and works, but it
constrains threading and makes unit-testing subsystems in isolation awkward. Accept it for
sim code; keep any new parallel work (pathfinding jobs) strictly on immutable snapshots.

### 5.5 The 4-bit tile type ceiling

16 tile types total. OpenTTD uses 11. If your design wants more distinct tile classes, plan
the widening in the same pass as §5.2 — retrofitting it later touches every `GetTileType`
call site and the save format.

### 5.6 You inherit 25 years of transport-shaped assumptions

Bridges/tunnels ("wormholes") leak into movement and tile Z logic. `Station` catchment,
cargo acceptance in eighths, town growth driven by passenger transport, `AddAcceptedCargo`
callbacks on tiles. Every one of these needs a decision: delete, or repurpose. Expect
archaeology. `docs/landscape.html` and the Doxygen output are your friends; read them
before touching the tile layer.

---

## 6. Gameplay-to-engine mapping

Concrete mapping from your design pillars to what exists:

| Your feature | OpenTTD basis | Verdict |
|---|---|---|
| Isometric TTD look | `viewport.cpp` + blitters + zoom levels | **Free.** The whole reason to fork. |
| Sloped terrain, terraforming | `landscape.cpp`, `terraform_cmd.cpp`, `Slope` | **Free**, minor edits |
| Map generation | `tgp.cpp`, `genworld.cpp`, `heightmap.cpp` | **Free**, retune for small maps + resource seeding |
| Forest growth / regrowth | `TileLoop_Trees` (`tree_cmd.cpp:836`) | **Adapt** — genuinely close to what you need |
| Crop fields, soil | `TileLoop_Clear` + `ClearGround::Fields` | **Adapt** as skeleton, rewrite the model |
| Buildings placed on tiles | `Object` tile type (`object_cmd.cpp`, 1,037 LOC) + `ObjectSpec` | **Best template** — multi-tile, sized, view-variant, availability-gated, minimal baggage |
| Production buildings | `Industry` (`industry.h:64`) | **Rework** — structure fits (location, produced/accepted lists, history), model doesn't (no labour) |
| Stockpiles / granaries | `Station` `GoodsEntry` | **Rewrite.** Don't force it; write a clean `Storage` pool |
| Villagers | `Vehicle` movement/render fields only | **Harvest ~15 fields, discard 105** |
| Villager pathfinding | — | **Build.** 6–10 weeks |
| Jobs / hauling | — | **Build.** 8–12 weeks. This is the game |
| Needs, families, lifecycle | — | **Build** |
| Tech tree | `Engine`/`ObjectSpec` availability gating pattern | **Build**, reuse the gating shape |
| Resource economy (no money) | `CargoPacket`, `Money`, `CommandCost` | **Neutralise money, rewrite resources** |
| Seasons / weather | Snow line, climate zones (cosmetic) | **Build** the gameplay layer |
| Save/load | `saveload/` chunked versioned serialiser | **Free** framework, new chunks |
| UI | `window.cpp`/`widget.cpp`/`widgets/` | **Free** framework, all new windows |
| Localisation | `strgen/`, `lang/` | **Free** |
| Sound/music | `sound/`, `music/`, mixer | **Free** |
| Multiplayer | `network/` lockstep | **Delete** unless you want it — and if you *might*, keep the command discipline |
| Modding | Squirrel VM (`3rdparty/squirrel`, `script/`) | **Delete the TTD API, keep the VM** — 158 API files are transport-specific, but the embedded Squirrel interpreter is a ready-made modding host |

---

## 7. Alternatives, honestly compared

You asked whether OpenTTD is viable. It is — but you should know what you're choosing
against.

| Base | Licence | Isometric TTD look | Free-roaming agent sim | Verdict for this project |
|---|---|---|---|---|
| **OpenTTD** | GPL v2 | **Exactly** — it *is* the look | None (track-based YAPF only) | Best renderer match, ships-with-free-art, 8bpp *and* 32bpp RGBA. You write all the agent/village sim. **Recommended.** |
| **OpenRCT2** | GPL v3 | Very close (same TTD lineage) | **No** — see below | **Evaluated in full: [04-openrct2-feasibility.md](04-openrct2-feasibility.md). Rejected.** Requires purchased RCT2 assets to run, and is hard-locked to 256 colours. Its "peeps" walk on footpath tiles only, via a deliberately-not-A\* one-step search. |
| **Widelands** | GPL v2+ | Different (Settlers-style, softer) | **Yes** — workers, wares, carriers, production chains, ware economy | Closest *gameplay* match by far — its economy is almost your economy. But the visual style is not TTD's, and that's your stated pillar. Worth a day if you'd trade the look for the sim. |
| **Unknown Horizons / FIFE** | GPL v2 / LGPL | Isometric, but Anno-style | Partial | Smaller, less active; Python performance ceiling for thousands of agents. |
| **Godot 4** | MIT | Build it yourself | Build it yourself | Permissive licence, modern tooling, great iteration. But you write the isometric sorted renderer *and* the sim. Realistically the largest total effort for a TTD look. |
| **From scratch (SDL/raylib)** | Yours | Build it yourself | Build it yourself | Maximum control, maximum cost. Only sane if the renderer is your core competency. |

**On OpenRCT2 — correcting an earlier version of this document.** This section previously
claimed OpenRCT2's peeps are *"individual agents with needs, thoughts, and pathfinding"* and
that forking it would let you inherit agent simulation along with the look, and recommended a
day's evaluation. The evaluation was done and **that claim was wrong.** Guests walk only on
placed footpath tiles; the pathfinder is an explicitly-not-A\* depth-first search returning one
direction per step; and the `Guest` struct's ~40 fields yield exactly three (`hunger`,
`Energy`, `happiness`) that transfer to a villager. On top of that it requires purchased
RollerCoaster Tycoon 2 files to run and cannot render more than 256 colours. Full analysis and
the four architectural patterns worth importing from it are in
[04-openrct2-feasibility.md](04-openrct2-feasibility.md).

**The trade in one sentence:** OpenTTD gives you the exact visual identity and the hardest
rendering problem solved, and gives you nothing toward the agent simulation. **No candidate
engine gives you agent pathfinding** — that is work you do regardless of base, so choose on
renderer, art pipeline, and licence instead.

---

## 8. Recommendation

**Proceed with the OpenTTD fork, conditional on the GPL v2 decision, and de-risk in this
order:**

1. **Resolve licensing first.** One conversation, before any code. If closed-source or
   console is required, switch bases now.
2. ~~Spend one day evaluating OpenRCT2.~~ **Done** — see
   [04-openrct2-feasibility.md](04-openrct2-feasibility.md). Rejected as a base; four
   architectural patterns imported into the rework plan instead.
3. **Build a vertical-slice spike before the big deletion.** Two to three weeks, on an
   unmodified fork: add one custom `TileTypeProcs` for a "forest" tile, add a lean
   `Villager` pool that free-moves in world coordinates using the disaster-vehicle
   precedent, write a naive grid A*, and have ten villagers walk to trees, chop, and haul
   wood to a stockpile. Ugly placeholder art is fine.

   This spike answers the three questions that actually decide the project:
   - Does the sprite sorter render your actors and buildings correctly on slopes?
   - Can you get 500–2,000 pathing agents inside a tick budget?
   - Is the codebase pleasant to work in *for you*?

   If the spike goes well, the rest is a long but well-understood grind. If it doesn't, you
   have spent three weeks instead of a year.
4. **Then** execute the [engine rework plan](02-engine-rework-plan.md).

**Honest expectations.** A fork does not make this a small project. It removes maybe 12–18
months of renderer, UI, serialisation, and localisation work — which is a lot — and leaves
you the entire simulation game to write. For a small team, plan in years, not months. The
fork's value is that you spend those years on *your* game's problems (villagers, needs,
jobs, tech) rather than on sprite occlusion sorting.

---

*Assessed against OpenTTD `be4f099` (2026-08-10), 403,957 LOC. Build verified on
Ubuntu 24.04 / GCC 13.3.*
