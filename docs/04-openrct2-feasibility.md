# Feasibility: OpenRCT2 as the base for a Lithic-era settlement game

**Same investigation as [the OpenTTD assessment](01-feasibility.md), applied to OpenRCT2
`4e576a0` (2026-08-10).** This was the one-day evaluation that assessment recommended.

**Verdict: OpenRCT2 is a worse base than OpenTTD for this project.** Not marginally — two
findings are close to disqualifying on their own, and the single reason I flagged it for
evaluation turns out not to exist.

**Correction to the OpenTTD assessment.** It said OpenRCT2's peeps are *"individual agents
with needs, thoughts, and pathfinding"* and that forking it would let you *"inherit agent sim
and the look."* Having now read the code: guests walk **only on footpath tiles**, the
pathfinder is an explicitly-not-A\* depth-first heuristic that returns one direction per step,
and the `Guest` struct is almost entirely park-specific. The agent-simulation advantage I
hypothesised is largely illusory. That claim in §7 of the OpenTTD assessment was wrong and
has been corrected there.

**But the evaluation was still worth the day**, because OpenRCT2 does four things
*architecturally better* than OpenTTD, and those are patterns worth importing into the
OpenTTD fork. See [§7](#7-what-to-steal-for-the-openttd-fork) — that's the actionable output.

---

## Contents

- [1. The two near-disqualifying findings](#1-the-two-near-disqualifying-findings)
- [2. Licensing](#2-licensing-gpl-v3)
- [3. What the codebase actually is](#3-what-the-codebase-actually-is)
- [4. The peep system, examined](#4-the-peep-system-examined)
- [5. Where OpenRCT2 is genuinely better than OpenTTD](#5-where-openrct2-is-genuinely-better-than-openttd)
- [6. Other frictions](#6-other-frictions)
- [7. What to steal for the OpenTTD fork](#7-what-to-steal-for-the-openttd-fork)
- [8. Head-to-head](#8-head-to-head)
- [9. Recommendation](#9-recommendation)

---

## 1. The two near-disqualifying findings

### 1.1 It requires the original RollerCoaster Tycoon 2 game files to run

From `readme.md:74`:

> OpenRCT2 **requires original files of RollerCoaster Tycoon 2** to play. It can be bought at
> either Steam or GOG.com.

This is categorically worse than OpenTTD's situation. OpenTTD needs *a* base graphics set, and
a complete, freely-redistributable, GPL-licensed one exists (OpenGFX) that you may ship. RCT2's
assets are proprietary Chris Sawyer / Atari property. The engine locates them at runtime:

```
src/openrct2/platform/Platform.Common.cpp:139   Path::Combine(path, u8"Data", u8"g1.dat")
src/openrct2/SpriteIds.h:1049                   SPR_RCTC_G1_END = 29357
```

`SpriteIds.h` is 2,225 lines mapping **29,357 sprite indices** onto the layout of the original
`g1.dat`. To ship anything, you must replace all of it. The project has an in-progress
community effort for exactly this — the readme links a `#open-graphics` Discord channel — and
the fact that it's still an ongoing effort after a decade tells you how large the job is.

Practically: you'd be replacing 29k baked sprite indices *and* the terrain/scenery/path object
sets before you could distribute a single build. OpenTTD's equivalent is ~1,192 `SPR_*`
constants and a shippable free base set.

### 1.2 The renderer is hard-locked to a 256-colour palette

```c
// src/openrct2/drawing/ColourPalette.h:32
constexpr auto kGamePaletteSize = 256u;
using GamePalette = std::array<BGRAColour, kGamePaletteSize>;
```

Sprites are `G1Element`s holding **palette indices**, optionally RLE-compressed in RCT2's
format (`G1Flag::hasRLECompression`). `ImageImporter` accepts PNG input but quantises it into
the fixed palette (`GetClosestPaletteIndex`, `CalculatePaletteIndex`). Even the OpenGL engine
renders indexed colour and resolves it in a shader (`ApplyPaletteShader.cpp`).

**There is no 32bpp sprite path anywhere in the codebase.**

OpenTTD, by contrast, ships full RGBA 32bpp blitters (`blitter/32bpp_*.cpp`, including
SSE2/SSSE3/SSE4 variants and a 40bpp animated-palette blitter). You choose 8bpp or 32bpp.

A 256-colour palette is a legitimate art direction — RCT2 looks lovely — but on OpenRCT2 it
isn't a choice, it's a permanent constraint, and escaping it means rewriting the entire
drawing layer. For a game whose visual identity is a stated pillar, having the option matters.

---

## 2. Licensing: GPL v3

`licence.txt` is GNU GPL **version 3**. Every file header confirms it.

Compared to OpenTTD's GPL v2, v3 adds:

- **Anti-tivoisation** — you cannot ship on hardware that prevents users running modified
  versions. Consoles and locked-down platforms move from "practically blocked" to "explicitly
  forbidden."
- **An express patent grant** from contributors, plus termination if you initiate patent
  litigation.
- **Broader "Corresponding Source"** — installation information must accompany
  user-modifiable products.

For an ordinary PC indie release the practical outcome is much the same as v2 (sell it freely,
publish the source). But v3 is strictly less flexible, and GPLv2-only code cannot be combined
with it — so a v3 base forecloses reusing anything from OpenTTD later, in either direction.

---

## 3. What the codebase actually is

**792,058 lines** including vendored third-party — nearly double OpenTTD's 403,957. But the
distribution is startling:

| Fate | LOC | Share |
|---|---:|---:|
| **Delete** — rides, coasters, track & vehicle painting | 411,385 | **51.9%** |
| **Delete/rework** — UI windows | 65,447 | 8.3% |
| Vendored third-party | 71,768 | 9.1% |
| **Rework** — peeps & entities | 21,087 | 2.7% |
| Keep — `actions/` (commands) | 21,411 | 2.7% |
| Keep — scripting (JavaScript/QuickJS) | 20,691 | 2.6% |
| Keep — world / tile elements | 16,990 | 2.1% |
| **Delete** — RCT1/RCT2/RCT12 import & Sawyer coding | 15,698 | 2.0% |
| Keep — core/util/math/profiling | 14,948 | 1.9% |
| Keep — audio/platform/config/scenes | 14,085 | 1.8% |
| Keep — drawing | 14,243 | 1.8% |
| **Delete** — park management/finance/scenario | 12,748 | 1.6% |
| Keep — object system | 11,155 | 1.4% |
| Keep — paint (non-track) | 11,083 | 1.4% |
| Keep — misc root | 10,798 | 1.4% |
| Keep — UI framework | 35,648 | 4.5% |
| Keep — interface core | 9,111 | 1.2% |
| **Delete** — network | 9,012 | 1.1% |
| Keep — localisation | 4,653 | 0.6% |

**Over half the codebase is rollercoaster geometry.** The largest files in the tree:

```
42,083  src/openrct2/ride/VehicleSubpositionData.cpp
21,027  src/openrct2/paint/track/coaster/CorkscrewRollerCoaster.cpp
20,640  src/openrct2/paint/track/coaster/TwisterRollerCoaster.cpp
20,428  src/openrct2/paint/track/coaster/SingleRailRollerCoaster.cpp
20,111  src/openrct2/paint/track/coaster/LatticeTriangleTrack.cpp
```

The reusable engine core is roughly **~180k LOC** — comparable to OpenTTD's ~130k keepable,
but reached through a much larger and more irrelevant codebase.

**I could not configure the build**, in the same environment where OpenTTD configured *and
fully compiled* with only `zlib`, `liblzma`, `libpng`, `freetype`, `fontconfig`:

```
-- Checking for module 'libcurl'
--   Package 'libcurl', required by 'virtual:world', not found
CMake Error: The following required packages were not found: - libcurl
```

Required: `libcurl`, `libzip` ≥1.0, `zlib`, `zstd`, `libpng` ≥1.6, `OpenSSL` ≥1.0, `ICU` ≥59
(uc), `freetype`, `fontconfig`, `SDL2`, `FLAC`, `ogg`, `vorbisfile`, `Threads`, OpenGL —
plus bundled QuickJS-ng, sfl, picosha2. That's a materially heavier dependency chain, and it
means a heavier onboarding and CI story.

---

## 4. The peep system, examined

This was the entire reason to evaluate OpenRCT2. It does not hold up.

### 4.1 Guests walk on footpaths only

`ChooseDirection` (`peep/GuestPathfinding.cpp:1224`) skips anything that isn't a path element:

```c
if (destTileElement->getType() != TileElementType::path)
    continue;
```

Guests cannot walk on grass, dirt, or open terrain. Movement is a graph over placed footpath
tiles. A Lithic village is villagers crossing open ground to reach a tree — the exact case
this system does not model.

### 4.2 The pathfinder is deliberately not a pathfinder

From the implementation's own documentation (`GuestPathfinding.cpp:658-666`):

> The implementation is a depth first search of the path layout in xyz according to the search
> limits. **Unlike an A\* search**, which tracks for each tile a heuristic score […] a single
> best result "so far" […] is tracked via the score parameter. With this approach, explicit
> loop detection is necessary to limit the search space, and each alternate route through the
> same tile can be returned as the best result, rather than only the shortest route with A\*.

It returns **one `Direction`** — the next step — not a path. There is no cached route, no flow
field, no reusable plan. Every step re-searches, bounded by:

```c
state.maxJunctions   = PeepPathfindGetMaxNumberJunctions(peep);
int32_t maxTilesChecked = (peep.is<Staff>()) ? 50000 : 15000;   // per query
```

And it faithfully preserves the original game's quirks, including the "wide path" rule where
guests refuse to path along paths wider than one tile, and a source comment resigned to it:

> Anyone attempting to overlay paths with different slopes should EXPECT to experience path
> finding irregularities due to those paths! Simply do not do it! :-)

For a village sim you would delete all 2,121 lines and write the layered pathfinder described
in [the rework plan §5](02-engine-rework-plan.md#phase-5--pathfinding) anyway. **Nothing is
saved.**

### 4.3 `Guest` is park-shaped, not person-shaped

From `entity/Guest.h:259`, the actual fields:

`guestNumRides`, `guestNextInQueue`, `guestHeadingToRideId`, `guestTimeOnRide`, `paidToEnter`,
`paidOnRides`, `paidOnFood`, `paidOnDrink`, `paidOnSouvenirs`, `nausea`, `nauseaTarget`,
`nauseaTolerance`, `intensity`, `timeInQueue`, `daysInQueue`, `photo1RideRef`…`photo4RideRef`,
`voucherRideId`, `favouriteRide`, `favouriteRideRating`, `amountOfSouvenirs`, `balloonColour`,
`umbrellaColour`, `hatColour`, `rejoinQueueTimeout`, `previousRide`, `vandalismSeen`…

And `Peep` (`entity/Peep.h:308`) adds `CurrentRide`, `CurrentRideStation`, `CurrentTrain`,
`CurrentCar`, `CurrentSeat`, `TimeToSitdown`, `MazeLastEdge`, `spiralSlideSubstate`,
`timesSlidDown`, `RideSubState`, `UsingBinSubState`…

Of roughly forty fields, the ones that transfer to a villager are **`hunger`, `Energy`,
`happiness`** — and the generic position/orientation base. That is not an agent-simulation
head start; it's a lean movement base plus three integers, which OpenTTD's `Vehicle` also
gives you.

### 4.4 Entity storage is hostile to thousands of lean agents

```c
// src/openrct2/entity/EntityRegistry.h:32
union Entity_t {
    uint8_t Pad00[0x200];      // 512 bytes
    EntityBase base;
};
...
Entity_t entities[kMaxEntities]{};   // kMaxEntities = 65535
```

**A fixed 512 bytes per entity slot, 65,535 slots, statically allocated — 33.5 MB regardless
of how many entities exist.** Every villager occupies a 512-byte slot in a 512-byte-strided
array. That's 4× the ≤128-byte budget the guidelines set for `Villager`, and iterating
thousands of them touches four times as many cache lines as necessary. The union is sized by
the largest entity — `Vehicle`, a rollercoaster car.

Fixable in a fork, but it's an inherited constraint where OpenTTD's slot-reusing `Pool`
template is simply better suited.

### 4.5 What *is* worth taking: the thoughts system

`std::array<PeepThought, kPeepMaxThoughts> thoughts` plus `peep/PeepThoughts.cpp` — peeps form
legible opinions ("I'm hungry", "the queue for X is too long", "this park is very clean") that
surface in the UI. It's the mechanism that makes RCT2's crowds feel like people rather than
particles, and it's *exactly* the right UX for a Banished-like game where the player needs to
diagnose why a settlement is failing.

That is a **design pattern worth copying**, not code worth porting — the implementation is
entirely about rides, queues, and litter.

---

## 5. Where OpenRCT2 is genuinely better than OpenTTD

Four of these are meaningful, and all four are importable.

### 5.1 The tile-element model is better for a building game

OpenTTD: one type per tile, 12 packed bytes, hence the whole tile-widening exercise in
[rework plan Phase 3](02-engine-rework-plan.md#phase-3--tile-model-rework).

OpenRCT2: a **variable-length list of 16-byte typed elements per tile**, terminated by an
`isLastForTile()` flag, drawn from a global pool of up to ~16.7 M elements
(`kMaxTileElements`), indexed by `TilePointerIndex`. Each element carries:

```c
// src/openrct2/world/tile_element/TileElementBase.h:53
struct TileElementBase {
    uint8_t type;
    uint8_t flags;            // upper nibble flags, lower nibble occupied quadrants
    uint8_t baseHeight;
    uint8_t clearanceHeight;
    uint8_t owner;
};
```

Concrete element types: `surface`, `path`, `track`, `smallScenery`, `largeScenery`, `wall`,
`entrance`, `banner`.

For a village that wants ground **plus** a building **plus** a wall **plus** a decoration on
one tile, at different heights, with quarter-tile occupancy — this is materially more
expressive than OpenTTD's single-type tile, and it's the model you'd want. `SurfaceElement`
even carries `GrassLength` with `UpdateGrassLength()`, i.e. grass growth is already modelled,
plus per-tile `WaterHeight`, `Slope`, and a swappable `SurfaceStyle` terrain object.

### 5.2 The asset pipeline is already modern

Objects are `.parkobj` archives: a zip containing `object.json` plus a PNG image table
(`ObjectFactory.cpp:452`, `ImageTable.cpp`, `ImageImporter.cpp`, `AssetPack.cpp`). 79 files of
object system, with 20+ typed object classes including `PeepAnimationsObject` — **peep
animations are themselves data-driven content objects**.

This is precisely what [OpenTTD rework Phase 2](02-engine-rework-plan.md#phase-2--graphics-pipeline-liberation)
proposes *building* (1–2 weeks). OpenRCT2 has it. Caveat: it imports into the 256-colour
palette (§1.2), so it's a better *pipeline* delivering a worse *ceiling*.

### 5.3 Replay and desync infrastructure already exists

- `ReplayManager.cpp` — 890 lines, record/replay of the command stream.
- `GameStateSnapshots.cpp` — 801 lines, per-field state comparison
  (`CompareSpriteDataPeep`, `CompareSpriteDataCommon`, …) that reports *which field on which
  entity* diverged.
- `EntitiesChecksum` — SHA-1 over all entity state.
- `assets.json` ships a versioned **repository of recorded replays** used in CI.

This is exactly the harness [guidelines §13](03-game-logic-guidelines.md#the-replay-harness)
calls the highest-value test infrastructure you can have — and OpenRCT2 built it, uses it, and
has the per-field diff tooling that turns "the hash diverged" into "`Guest::hunger` differed on
entity 4,102 at tick 91,338." That diagnostic step is the expensive part.

### 5.4 `EntityTweener` — interpolated rendering

`entity/EntityTweener.cpp` (188 lines) interpolates entity positions between simulation ticks,
so visual smoothness is decoupled from tick rate. OpenTTD has no equivalent. For a game
rendering hundreds of walking people at 60 fps on a 30 Hz sim, this is a real quality
difference and it's a small, self-contained piece of code.

### 5.5 Smaller wins

- **`GameAction` class hierarchy** (`actions/`, 179 files across 10 categories) with
  Query/Execute plus `Serialise()` — cleaner and more type-safe than OpenTTD's function-pointer
  command table. `actions/terraform/` (18 files) and `actions/scenery/` (34) are directly
  relevant to a building game.
- **JavaScript plugins** via bundled QuickJS-ng (20,691 LOC, 70 files) — a far more
  approachable modding host than Squirrel.
- **Finer movement resolution**: `kCoordsXYStep = 32` world units per tile vs OpenTTD's 16.
- **Map generator**: `SimplexNoise.cpp`, `PngTerrainGenerator.cpp`, `TreePlacement.cpp`,
  `MapHelpers` — smaller than OpenTTD's `tgp.cpp` but cleanly structured.
- **Per-tile spatial index for entities** (`gEntitySpatialIndex`) alongside the viewport index.

---

## 6. Other frictions

- **Paint sorting is quirkier.** OpenRCT2 buckets paint structs by quadrant and sorts within
  (`PaintStructsSortQuadrantLegacy`, `gPaintStableSort` in `paint/Paint.cpp`), with a
  `Legacy` mode explicitly preserving original-game sorting bugs. It works, and quadrant
  bucketing may well be faster — but OpenTTD's global topological bounding-box sort
  (`ViewportSortParentSprites`, with an SSE4.1 path) is cleaner and has no bug-compatibility
  mandate. For an art style you're inventing, you want the sorter without inherited quirks.
- **Map size caps at 1001×1001** (`kMaximumMapSizeTechnical`). Ample for this game, but worth
  knowing it's a hard limit rather than OpenTTD's 4096×4096.
- **Deep RCT12/RCT2 legacy layer** — `rct1/`, `rct12/`, `rct2/`, `sawyer_coding/` (15,698 LOC)
  plus `RCT12::Limits` reaching into `Limits.h`. Save format, object format, and many constants
  are shaped by 1999 file layouts. Excising this is real archaeology.
- **792k LOC to navigate** when half is irrelevant. Code quality is good — modern C++20,
  namespaced, consistent — but the signal-to-noise ratio for your purposes is poor.

---

## 7. What to steal for the OpenTTD fork

The actionable output of this evaluation. Four amendments to
[the rework plan](02-engine-rework-plan.md):

| Import | Into | Why |
|---|---|---|
| **Variable-length tile-element lists** with `baseHeight`/`clearanceHeight`/occupied-quadrants | Phase 3 (tile model) | Replaces the plan's "widen the tile struct" with something strictly better: ground + building + wall + decoration stacking on one tile, at different heights, with quarter-tile occupancy. Adopt the *model*, not the code. |
| **`.parkobj`-style content objects** — zip of `object.json` + PNG image table, with animation sets as data objects | Phase 2 (PNG pipeline) | A validated design for the loader you're writing anyway. Note OpenTTD lets you keep full RGBA, so you get the better pipeline *and* the better ceiling. |
| **Per-field state-diff replay tooling** | Phase 13 (replay harness) | The plan already calls for record/replay; OpenRCT2 shows the piece that makes it usable — reporting *which field on which entity* diverged, not just that a hash did. |
| **Entity tweening** | Phase 4 (actor substrate) | ~200 lines, decouples visual smoothness from tick rate. Cheap, high perceived-quality return. |

Two design patterns worth copying regardless of base:

- **The thoughts system** (§4.5) — legible per-agent opinions surfaced in the UI. The right
  answer to "why is my settlement dying?"
- **Class-based commands with `Serialise()`** — more type-safe than function-pointer tables,
  and serialisation falls out for free, which the replay harness needs.

One thing to explicitly *not* copy: OpenRCT2's fixed-size 512-byte entity union. OpenTTD's
`Pool` template is the better primitive.

---

## 8. Head-to-head

| | **OpenTTD** | **OpenRCT2** |
|---|---|---|
| Licence | GPL v2 | GPL **v3** (stricter; anti-tivoisation) |
| Total LOC | 403,957 | 792,058 |
| Irrelevant bulk | ~41% (transport) | **~52% (rollercoaster geometry alone)** |
| Reusable core | ~130k | ~180k |
| **Shippable art possible?** | **Yes** — free GPL base set (OpenGFX) | **No** — requires purchased RCT2 files; ~29,357 baked sprite IDs |
| **Colour depth** | **8bpp *and* full RGBA 32bpp** | **256-colour palette only, no 32bpp path** |
| Sprite sorting | Global topological bbox sort, SSE4.1 path, clean | Quadrant-bucketed, `Legacy` bug-compat mode |
| Tile model | 1 type/tile, 12 packed bytes | **Variable-length 16-byte element list, height-stacked** |
| Asset pipeline | GRF containers (needs replacing) | **JSON + PNG objects (already modern)** |
| Entity storage | **Slot-reusing `Pool`, lean structs** | Fixed 512 B × 65,535 = 33.5 MB static |
| Movement resolution | 16 units/tile | **32 units/tile** |
| Interpolated rendering | No | **Yes (`EntityTweener`)** |
| Replay/desync tooling | Command pattern only; harness to build | **Built, with per-field diffs + CI replay corpus** |
| Scripting | Squirrel | **JavaScript (QuickJS-ng)** |
| Commands | Function-pointer table | **Class hierarchy w/ `Serialise()`** |
| Map size cap | 4096×4096 | 1001×1001 |
| Build dependencies | 5 common libs; **built clean in 524 s** | 15+ incl. libcurl/libzip/zstd/ICU/SDL2; **would not configure** |
| Free-roaming agent pathfinding | None (track-based YAPF) | **None (footpath-only, not-A\*, one-step)** |
| Agent needs/lifecycle | None | 3 usable fields of ~40; rest is rides/nausea/queues |
| Legacy baggage | Old savegame loader | `rct1/`+`rct12/`+`rct2/`+`sawyer_coding` (15.7k) |

**Neither engine gives you agent pathfinding.** That was the whole hypothesis, and it fails for
both. The comparison therefore reduces to: which renderer, art pipeline, and tile model do you
want to build the simulation on top of — and OpenTTD wins on the two that can't be fixed later
(shippable assets, colour depth) while losing on two that can be imported (tile model, asset
pipeline).

---

## 9. Recommendation

**Stay with the OpenTTD fork.** The decision turns on two constraints that a fork cannot
engineer away:

1. **You cannot ship an OpenRCT2-based game without replacing ~29,357 proprietary sprite
   indices first.** OpenTTD lets you start from a free, redistributable base set and replace
   art incrementally. That difference alone reorders the two options.
2. **OpenRCT2 can never render more than 256 colours.** OpenTTD hands you full RGBA 32bpp
   blitters and an OpenGL backend, and *lets you choose* a palettised look if you want one.
   For a project whose visual identity is a stated pillar, keeping the ceiling open is worth
   more than any of OpenRCT2's advantages.

The GPL v3 question is secondary but real: it further narrows commercial and platform options,
and it forecloses ever borrowing from OpenTTD's GPLv2-only renderer.

**Amend the plan rather than the base.** Fold the four imports in
[§7](#7-what-to-steal-for-the-openttd-fork) into the existing phases — the tile-element model
into Phase 3, the JSON+PNG object format into Phase 2, per-field replay diffs into Phase 13,
entity tweening into Phase 4. The tile-element import is the most valuable of the four and
genuinely improves the plan: it replaces "widen the packed tile struct and document the bit
layout" with a variable-length height-stacked element list, which is both more expressive and
easier to keep documented.

**One thing this investigation does not change:** the
[Phase 0 vertical spike](02-engine-rework-plan.md#phase-0--fork-hygiene--vertical-spike) is
still the right next step, and its questions are unchanged. Both candidate engines lack agent
pathfinding, so "can 2,000 pathing villagers fit the tick budget on this renderer" remains the
question that decides the project — and it's now clear you must answer it with a pathfinder you
write yourself, on whichever base you pick.

---

*Assessed against OpenRCT2 `4e576a0` (2026-08-10), 792,058 LOC. CMake configure attempted on
Ubuntu 24.04 / GCC 13.3; failed on missing `libcurl` before reaching compilation.*
