# Engine rework plan

Turning OpenTTD `be4f099` into an engine for a Lithic-era settlement game.

Read [the feasibility assessment](01-feasibility.md) first — this plan assumes its
conclusions, notably that the tile-type dispatch table is the main extension seam and that
the GRF-bound graphics pipeline must be replaced early.

**Amendments from the OpenRCT2 evaluation.** [That assessment](04-openrct2-feasibility.md)
rejected OpenRCT2 as a base but identified four architectural patterns worth importing. They
are folded into the phases below and flagged **[from OpenRCT2]** where they appear:

| Import | Phase | Effect on the plan |
|---|---|---|
| Variable-length tile-element lists | 3 | **Replaces** the "widen the packed tile struct" approach with something better |
| `.parkobj`-style JSON+PNG content objects | 2 | Validated format for the loader you're writing anyway |
| Entity tweening | 4 | ~200 lines; decouples visual smoothness from tick rate |
| Per-field state-diff replay tooling | 13 | Turns "the hash diverged" into an actionable field name |

## Guiding principles for the whole rework

1. **Never break the build for more than a day.** The fork must stay runnable at every
   commit. A six-week "great refactor" branch that doesn't compile is how these projects
   die.
2. **Delete before you build.** Every system you leave in place is a system you'll
   accidentally couple to. Amputation is cheap now and expensive later.
3. **Two amputation passes, not one.** Cut the obviously-unreachable transport code first
   (Phase 1), then the economy after you know what replaces it (Phase 6). Trying to do both
   at once means holding the whole design in your head while the compiler screams.
4. **Change the data model before the gameplay.** Tile layout and the actor struct are the
   foundations. Reworking them after gameplay exists means reworking gameplay too.
5. **Keep the command/determinism discipline** even though you're single-player. It buys
   replay testing, which you will need more than you expect.
6. **Spike before you commit.** Phase 0 exists to answer questions, not to produce code you
   keep.

## Phase overview

| # | Phase | Duration | Gate: done when… |
|---|---|---|---|
| 0 | Fork hygiene & vertical spike | 3–4 wk | Villagers walk, chop, haul on an unmodified fork |
| 1 | Amputation I: transport | 3–4 wk | Rail/road/air/vehicle/NewGRF/network gone; game boots and scrolls |
| 2 | Graphics pipeline liberation | 2–3 wk | Game runs on PNG assets, zero GRF dependency |
| 3 | Tile model rework | 3–4 wk | New `TileType` set, widened tile struct, documented bit table |
| 4 | Actor substrate | 3–4 wk | `Villager` pool renders, moves, saves, scales to 2,000 |
| 5 | Pathfinding | 6–10 wk | 2,000 agents pathing inside tick budget, slope-aware |
| 6 | Amputation II: economy → resources | 4–6 wk | Money neutralised; `Storage` pool; resource ledger |
| 7 | Work & jobs | 8–12 wk | Full gather → haul → build → produce loop |
| 8 | Buildings & production | 5–7 wk | Data-driven building specs, construction sites, workplaces |
| 9 | Population lifecycle | 4–6 wk | Needs, families, birth/death, food/warmth pressure |
| 10 | Technology tree integration | 3–4 wk | Data-driven tech graph gates buildings/recipes/actions |
| 11 | UI layer | 6–8 wk | Settlement dashboard, villager inspector, build menu, research screen |
| 12 | World generation & scenario | 3–4 wk | Lithic-appropriate maps with seeded resources |
| 13 | Save/load consolidation | 2–3 wk | Versioned format, forward-compat policy, replay harness |
| 14 | Modding surface *(optional)* | 4–6 wk | Squirrel API for content |

Durations assume one experienced full-time C++ developer. They are sequential-dependency
estimates, not a promise; Phases 5/7 are the ones that slip.

---

## Phase 0 — Fork hygiene & vertical spike

**Purpose:** answer the three questions that decide the project before spending real money.

### 0.1 Repository setup

```bash
git clone https://github.com/OpenTTD/OpenTTD.git lithic
cd lithic
git remote rename origin upstream          # keep it: you'll want upstream renderer fixes
git remote add origin <your-repo>
git checkout -b main
git tag fork-point-be4f099                 # permanent record of the fork base
```

Keep `upstream` forever. Renderer, blitter, and GUI bugfixes land there and remain
cherry-pickable for years if you don't gratuitously reformat those files. **Corollary: touch
`viewport.cpp`, `blitter/`, `window.cpp`, and `widget.cpp` as little as possible.** Prefer
adding new files over editing shared ones.

### 0.2 Legal and identity hygiene

- Preserve `COPYING.md` and all existing per-file GPL headers. Do not strip them.
- Add `CREDITS-OpenTTD.md` acknowledging the upstream project and its authors.
- Rename the project, binary, config directory, and window title away from "OpenTTD"
  (trademark, and it prevents config collisions with a user's real OpenTTD install).
- Add your own copyright line to files you substantially author, keeping the GPL v2 notice.
- Record the licence decision from feasibility §1 in `docs/LICENSING-DECISION.md`.

### 0.3 Build and CI

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug -DOPTION_USE_ASSERTS=ON
cmake --build build -j$(nproc)
```

Verified working on Ubuntu 24.04 / GCC 13.3 with `zlib`, `liblzma`, `libpng`, `freetype`,
`fontconfig`. For a graphical build add `libsdl2-dev`. Set up CI on day one: Linux + Windows
+ macOS, Debug with asserts, and run `regression/` until you delete it.

Add `ccache` and consider `-DCMAKE_UNITY_BUILD` off for incremental work. The single
`openttd_lib` target means a widely-included header change rebuilds everything — put your
new headers where they aren't transitively included by `stdafx.h`.

### 0.4 The vertical spike

**On the unmodified fork**, in new files only, minimal edits to existing ones:

| Step | What | Touches |
|---|---|---|
| 1 | `TileType::Forest` in the free 12th slot; a `_tile_type_forest_procs` with draw + tile_loop | new `forest_cmd.cpp`, `landscape.cpp:69`, `tile_type.h` |
| 2 | `Villager` pool item: `x_pos/y_pos/z_pos`, `direction`, `coord`, viewport-hash pointers, a `state` enum | new `villager.h/.cpp` |
| 3 | Render villagers by copying the `ViewportAddVehicles` hash-scan pattern (`vehicle.cpp:1149`) with any placeholder sprite | new `villager_gfx.cpp` |
| 4 | Free movement in world coords using `disaster_vehicle.cpp` as the precedent, with `GetSlopePixelZ()` for ground height | `villager.cpp` |
| 5 | Naive grid A* over `IsTileWalkable()` | new `pathfinder_agent.cpp` |
| 6 | Hardcoded loop: idle → nearest forest tile (via `core/kdtree.hpp`) → chop → haul to a fixed tile → repeat | `villager.cpp` |
| 7 | Spawn 10, then 500, then 2,000. Profile with the built-in framerate window | — |

**Spike exit criteria — all three must pass:**

- Villagers and multi-tile buildings sort correctly against terrain on slopes, with no
  visible occlusion glitches.
- 2,000 pathing agents fit in the tick budget (see [budgets](03-game-logic-guidelines.md#4-performance-budgets)).
- You find the codebase workable.

**Throw the spike code away afterward.** Its output is knowledge, plus a benchmark you keep.

---

## Phase 1 — Amputation I: transport

**Goal:** remove ~135k LOC of transport machinery. The game must still boot, generate a
map, scroll, terraform, and save.

### Order of deletion (dependency-driven — this order minimises breakage)

1. **`network/`** (~23.6k) — self-contained. Delete the directory, strip
   `NetworkSendCommand` from `command.cpp`, remove network settings and GUIs. Do this first;
   it's the cleanest win and it teaches you the deletion workflow.
2. **`script/`, `ai/`, `game/`** (~27.9k) — delete the 158 transport-specific API files and
   the AI/GameScript hosts. **Keep `3rdparty/squirrel`** — the VM is your future modding
   host (Phase 14). Delete `bin/ai`, `bin/game`, and the AI/GS settings GUIs.
3. **`newgrf*`** (~25.7k, 87 files) — the largest single win and the most tangled.
   `newgrf_*` hooks reach into houses, industries, stations, objects, canals, airports, and
   cargo. Work outward: delete the feature-specific `newgrf_<x>.cpp` alongside its owning
   subsystem, and stub the callback query points to constants before deleting the
   resolvers. Expect this to take the longest of the four.
4. **Air** (~6.8k) — `aircraft*`, `airport*`. Fewest dependents.
5. **Ship/water infra** (~4.5k) — ships, docks, locks, canals, `linkgraph/`'s water regions,
   `pathfinder/water_regions.*`. **Keep `TileType::Water` and `water_map.h`** — you need
   rivers, lakes, and coastline for fishing.
6. **Road/tram** (~14.9k) — `road*`, `tram*`, level crossings. **Note:** you may want to
   resurrect a much simpler version later for dirt paths that speed villager movement.
   Delete now, reimplement clean in Phase 3; don't try to keep it.
7. **Rail** (~22.1k) — `rail*`, `train*`, `signal*`, `track*`, `depot*`, `elrail`, PBS.
8. **Vehicles** (~42k) — `vehicle*`, `engine*`, `order*`, `group*`, `autoreplace*`,
   `articulated_vehicles`, `timetable`, `livery`, `consist`. **Harvest first** (see below).
9. **Bridges & tunnels** — `tunnelbridge*`, and the "wormhole" special cases that leak into
   Z-height and movement logic. Grep `EnteredWormhole` and follow it. If you want bridges
   later, reimplement rather than preserve.
10. **`pathfinder/yapf/`** — nothing salvageable for agents.
11. **`regression/`** and the vehicle/station/order GUIs.

### Harvest before deleting

Copy these into new files *before* the delete lands, or you'll be spelunking git history:

| From | What | For |
|---|---|---|
| `vehicle.cpp:1149` `ViewportAddVehicles` | Viewport spatial-hash scan | Actor rendering |
| `vehicle.cpp` | `_vehicle_viewport_hash`, `UpdateViewportPosition`, `coord` maintenance | Actor rendering |
| `vehicle_base.h` | `x_pos/y_pos/z_pos`, `direction`, `subspeed`, `progress`, `sprite_cache`, `MutableSpriteCache` | `Villager` |
| `disaster_vehicle.cpp` | `GetNewVehiclePos` free-movement pattern | Agent movement |
| `effectvehicle.cpp` | Short-lived visual effects | Smoke, dust, particles |
| `vehicle.cpp` | Tick dispatch + `CallVehicleTicks` amortisation | Actor tick loop |
| `object_cmd.cpp` | Multi-tile placement, footprint validation, `ObjectSpec` availability | Building placement |

### Keep deliberately

`core/`, `misc/`, `timer/`, `video/`, `blitter/`, `spriteloader/` (about to be extended),
`sound/`, `music/`, `gfx*`, `spritecache*`, `palette*`, `viewport*`, `landscape*`, `tile_*`,
`map_*`, `slope*`, `terraform*`, `clear_*`, `tree_*`, `water_map*`, `void_*`, `object_*`,
`genworld*`, `tgp*`, `heightmap*`, `smallmap*`, `window*`, `widget*`, `widgets/`, `strings*`,
`lang/`, `strgen/`, `settings*`, `saveload/`, `command*`, `console*`, `fileio*`, `fios*`,
`screenshot*`, `framerate_gui*`, `animated_tile*`.

### Gate

Game boots to a generated map, viewport scrolls at all 6 zoom levels, terraforming works,
save/load round-trips, no crashes, asserts clean. Commit as `v0.1-amputated`.

---

## Phase 2 — Graphics pipeline liberation

**Do this before commissioning any art.** It determines your artists' entire workflow.

### 2.1 `SpriteLoaderPNG`

`SpriteLoader` is a clean virtual interface (`spriteloader/spriteloader.hpp:100`), and
`ReadSprite()` (`spritecache.cpp:454`) is the single place that hardcodes
`SpriteLoaderGrf`. Add an implementation that reads PNG atlases plus a manifest.

**[from OpenRCT2]** Model the container on OpenRCT2's `.parkobj` format: a zip archive holding
an `object.json` manifest plus a PNG image table, loaded by an object factory
(`ObjectFactory.cpp`, `ImageTable.cpp`, `ImageImporter.cpp`). It is a proven design for exactly
this job, it keeps a building's spec and its art in one shippable unit, and OpenRCT2 goes
further by making *animation sets* data objects too (`PeepAnimationsObject`) — worth copying
for villager animation variety. The one thing not to copy: OpenRCT2 quantises imported PNGs
into a 256-colour palette. On OpenTTD you keep full RGBA, so you get the better pipeline *and*
the better ceiling.

Manifest sketch (JSON — `3rdparty/nlohmann` is already vendored):

```json
{
  "atlas": "gfx/buildings.png",
  "sprites": [
    { "name": "building/longhouse/n",   "rect": [0, 0, 96, 80],
      "offset": [-48, -56], "zoom": "normal",
      "bounds": { "origin": [0,0,0], "extent": [32,32,40] } },
    { "name": "building/longhouse/n",   "rect": [96, 0, 192, 160],
      "offset": [-96, -112], "zoom": "in2x" }
  ]
}
```

Requirements:
- Full RGBA (the 32bpp blitters support it — you are not confined to a 256-colour palette).
- Optional `m` remap channel per pixel for recolouring (clothing, seasonal tint) — the
  `CommonPixel` struct (`spriteloader.hpp`) already carries `r,g,b,a,m`.
- Per-zoom variants keyed to `ZoomLevel`, with automatic downscaling as a fallback so
  artists only author `normal` and `in2x`.
- Hot-reload in debug builds. This alone is worth the phase.

### 2.2 Named sprite registry

Replace the 1,801-line hardcoded `table/sprites.h` ID table with a symbolic registry:

```cpp
SpriteID GetSprite(std::string_view name);           // cached, asserts in debug on miss
constexpr SpriteRef LONGHOUSE_N{"building/longhouse/n"};  // resolved once at load
```

Keep numeric `SpriteID` internally — the cache and blitters depend on it — but never write
one in game code. Resolve names once at load into a lookup table; game code holds indices.

### 2.3 Bootstrap removal

Delete `bootstrap_gui.cpp` and the base-set download machinery. Your art ships with the
game. Keep `BaseGraphics`/`GraphicsSet` only if you want swappable art sets; otherwise
collapse it to a single asset root.

### 2.4 Fixed art specification

Lock these before art production and write them down:

- Tile footprint: `TILE_PIXELS = 32` at `ZoomLevel::Normal` → 64×32 diamond. Do not change
  this; it's woven through the viewport maths.
- Height step: `TILE_HEIGHT = 8` pixels per level.
- Max structure height: `MAX_BUILDING_PIXELS = 200` (raise if you want tall totems/trees).
- Max actor sprite: `MAX_VEHICLE_PIXEL_X/Y = 192/96` — rename, and shrink for villagers
  (smaller values shrink the viewport hash scan margin, so this is a real perf win).
- 8 facing directions (`Direction::End == 8`) — matches OpenTTD's `Direction` enum, so
  reuse it rather than inventing your own.

### Gate

Game runs with zero `.grf` files present. An artist can add a building by dropping a PNG and
editing a manifest, with hot-reload in debug. Commit as `v0.2-png-pipeline`.

---

## Phase 3 — Tile model rework

**This phase changed as a result of the [OpenRCT2 evaluation](04-openrct2-feasibility.md).**
The original plan widened OpenTTD's fixed 12-byte, one-type-per-tile struct. OpenRCT2's model
is better for a building game, and adopting it is the single most valuable import from that
evaluation. Both options are documented; pick one deliberately, because this is the decision
you cannot cheaply revisit.

### 3.1 Choose the tile representation

#### Option A (recommended) — variable-length tile-element lists **[from OpenRCT2]**

Replace "one type per tile" with a per-tile **list of typed 16-byte elements**, allocated from
a global pool and terminated by a last-for-tile flag, indexed by a per-tile pointer array
(OpenRCT2's `TileElementBase` / `TilePointerIndex` / `kMaxTileElements`):

```cpp
struct TileElementBase {       // 16 bytes total per element
	uint8_t  type;             // Surface / Ground / Building / Wall / Prop / Path / Resource
	uint8_t  flags;            // last-for-tile, ghost, invisible, occupied quadrants
	uint8_t  base_height;      // bottom of this element
	uint8_t  clearance_height; // top — what can stack above it
	uint16_t owner_ref;        // Building/Storage pool index
	/* + 10 bytes interpreted per element type */
};
```

Why this wins for a Lithic village:

- **Vertical stacking.** Ground + building + a wall + a decoration on one tile, each at its own
  height, with `clearance_height` making "can I build above this?" a local check rather than a
  special case.
- **Quarter-tile occupancy** via the flags nibble — small props (a drying rack, a firepit, a
  boulder) share a tile without consuming it.
- **No bit-packing arms race.** Each element type owns its own 10 bytes. Adding a field to
  buildings doesn't compete with forestry for space, which is the structural reason OpenTTD's
  tile layer is painful to extend.
- **Sparse cost.** Empty terrain is one surface element; complexity costs memory only where it
  exists.

Costs, honestly: an extra indirection on every tile access (mitigated by the pointer index),
a pool allocator with insert/remove/defragment, and a rewrite of every `GetTileType(t)` call
site rather than a mechanical widening. It also means diverging further from upstream's
`landscape.cpp` — keep the changes surgical so viewport fixes stay cherry-pickable.

#### Option B (simpler) — widen the flat struct

```cpp
// map_func.h — replace TileBase/TileExtended
struct TileBase {          // was 8 bytes
	uint8_t  type;         // 5 bits type + flags  (see 3.2)
	uint8_t  height;
	uint16_t owner_ref;    // Building/Storage/Zone pool index
	uint8_t  ground;       // ground kind + density
	uint8_t  flags;        // walkable, reserved, water-adjacent, indoors, snow
	uint16_t resource;     // quantity of the tile's resource (wood/stone/berries)
};
struct TileSim {           // new, 8 bytes — pure simulation state
	uint8_t  fertility;
	uint8_t  growth;       // crop/tree growth stage
	uint8_t  wear;         // path wear → movement cost
	uint8_t  snow_depth;
	uint16_t claim;        // agent/task reservation, INVALID = free
	uint16_t reserved;
};
```

16 bytes/tile = 4 MB on 512×512, 1 MB on 256×256. Trivial memory cost, far less work than
Option A, and adequate *if* your design never wants two constructed things on one tile. Decide
that now — retrofitting Option A after Phase 8 means reworking every building in the game.

**Recommendation:** Option A if buildings, walls, and props are a meaningful part of the
settlement's visual density (which, for a Banished-like, they usually are). Option B if you
want to reach a playable slice fastest and can accept one structure per tile.

#### Rules that apply to either option

**Non-negotiable process rule:** maintain `docs/tile-layout.md` as the authoritative
field/bit-allocation table, updated in the same commit as any layout change. OpenTTD's tile
layer is hard to modify precisely because this document (`docs/landscape.html`) drifted from an
exhaustively-packed reality. Don't repeat it. Prefer named fields over bit-packing until memory
actually forces your hand — at 512×512 it never will.

**Split hot from cold.** Keep render-hot data (touched every frame) separate from
simulation-only data (touched by the tile loop), as upstream already does with
`TileBase`/`TileExtended`. Real cache win either way.

**Steal `SurfaceElement`'s ideas** regardless of option: OpenRCT2's surface carries
`GrassLength` with an `UpdateGrassLength()` growth model, per-tile `WaterHeight`, `Slope`, and a
swappable `SurfaceStyle` terrain object. All four map directly onto what you need.

### 3.2 New tile type set

Under Option B, widen `TILE_TYPE_BITS` from 4 to 5 (32 slots) while you're here — retrofitting
later touches every `GetTileType` site and the save format. Under Option A the type lives in
the element and the ceiling is per-element-type, so this constraint disappears.

| Type | Notes |
|---|---|
| `Clear` | grass / dirt / rough / rock ground, via `ground` |
| `Water` | keep upstream's water/shore logic; add river flow if fishing needs it |
| `Woodland` | adapted from `TileType::Trees` — species, count, growth stage, density |
| `RockOutcrop` | quarryable stone/flint, finite `resource` |
| `Forage` | berries, roots, reeds — regrowing, seasonal |
| `Field` | tilled crop land — adapted from `ClearGround::Fields` |
| `Path` | worn trail; reduces movement cost; wears in/out by traffic |
| `Building` | all structures; `owner_ref` → `Building` pool |
| `Storage` | stockpiles, granaries, drying racks |
| `Construction` | site under construction; holds delivered materials + progress |
| `Void` | keep — map border sentinel, deleting it breaks bounds assumptions |

### 3.3 One `TileTypeProcs` per type

Implement `draw_tile_proc`, `tile_loop_proc`, `get_slope_pixel_z_proc`, `clear_tile_proc`,
`get_tile_desc_proc`, `terraform_tile_proc` per type and register in `_tile_type_procs`
(`landscape.cpp:69`). Use `clear_cmd.cpp` and `tree_cmd.cpp` as reference implementations.

Prune the `TileTypeProcs` struct itself: `get_tile_track_status_proc`,
`vehicle_enter_tile_proc`, `add_accepted_cargo_proc`, `add_produced_cargo_proc` and
`check_build_above_proc` are transport concepts. Replace with what you need:
`get_walk_cost_proc`, `get_harvest_yield_proc`.

### 3.4 Walkability and movement cost

```cpp
bool     IsTileWalkable(TileIndex tile);            // terrain + buildings + water
uint16_t GetTileWalkCost(TileIndex tile);           // ground kind + wear + snow
uint16_t GetTraversalCost(TileIndex from, TileIndex to);  // + slope penalty via GetSlopePixelZ
```

`GetTraversalCost` is the single hot function of the pathfinder. Design it now, keep it
branch-light, and make it a pure function of tile data so the pathfinder can run on an
immutable snapshot.

### 3.5 Dirty-flag protocol

Any change to walkability, height, or building footprint must publish an invalidation so the
pathfinder drops affected cached paths and flow fields. Define one choke point:

```cpp
void NotifyTileTraversabilityChanged(TileArea area);
```

Call it from every terraform, build, and demolish command. Getting this wrong produces
villagers walking through walls — and it is much easier to enforce now than to retrofit.

### Gate

New tile types render, tile loops run, terraforming and save/load work, `docs/tile-layout.md`
matches the code, and the chosen representation is recorded with its rationale in
`docs/decisions/`. Commit as `v0.3-tile-model`.

---

## Phase 4 — Actor substrate

### 4.1 The `Villager` pool

```cpp
using VillagerID = PoolID<uint16_t, struct VillagerIDTag, 0xFFFE, 0xFFFF>;

struct Villager : VillagerPool::PoolItem<&_villager_pool> {
	// --- position & rendering (harvested from Vehicle) ---
	TileIndex tile;
	int32_t   x_pos, y_pos, z_pos;
	Direction direction;
	mutable Rect coord;                       // NOSAVE viewport bounding box
	Villager *hash_viewport_next; Villager **hash_viewport_prev;   // NOSAVE
	mutable MutableSpriteCache sprite_cache;  // NOSAVE

	// --- identity & lifecycle ---
	uint16_t name_id;
	uint8_t  age_years, sex, health;
	FamilyID family;

	// --- needs (0..255 each) ---
	uint8_t hunger, warmth, rest, happiness;

	// --- work ---
	Profession profession;
	uint8_t    skill[NUM_PROFESSIONS];
	TaskID     current_task;
	BuildingID workplace, home;

	// --- carried ---
	ResourceType carried_type; uint16_t carried_amount;

	// --- movement (hot; keep compact) ---
	PathHandle path;
	uint16_t   path_step;
	uint8_t    speed, progress;
	VillagerState state;
};
```

Target **≤ 128 bytes**. At 2,000 villagers that's 256 KB — fits comfortably in L2, which
matters because you iterate all of them every tick. Keep needs/work/movement fields adjacent
so the hot tick loop touches few cache lines. Put anything rare (full name string, life
history, log) in a side table keyed by `VillagerID`, not inline.

**Counter-example worth knowing.** OpenRCT2 stores entities as a fixed 512-byte union across
65,535 statically-allocated slots (`union Entity_t { uint8_t Pad00[0x200]; ... }`) — 33.5 MB
reserved regardless of population, sized by its largest entity type, and iterated with a
512-byte stride. Do not do this. OpenTTD's slot-reusing `Pool` with a lean struct is the right
primitive, and the 128-byte target is what makes iterating thousands of agents cheap.

### 4.1b Interpolated rendering **[from OpenRCT2]**

Port the equivalent of OpenRCT2's `EntityTweener` (`entity/EntityTweener.cpp`, 188 lines):
cache each actor's pre- and post-tick position, and interpolate between them when drawing.
This decouples visual smoothness from the simulation tick rate, so 30 Hz sim renders as
smooth motion at 60+ fps. OpenTTD has no equivalent — its actors snap per tick, which is
unnoticeable on a train and very noticeable on a walking person.

Small, self-contained, and high perceived-quality return. Note the discipline it requires: the
interpolated position is **presentation-only state** and must never feed back into the
simulation ([guidelines §5](03-game-logic-guidelines.md#5-simulation-and-presentation-are-separate)).

### 4.2 Rendering

Port `ViewportAddVehicles` (`vehicle.cpp:1149`) verbatim as `ViewportAddVillagers`. It's a
bucketed spatial hash scan that only considers actors near the visible rect. Shrink the
`MAX_VEHICLE_PIXEL_X/Y` margins to villager-sized values — this directly reduces the scan
area.

Emit each villager via `AddSortableSpriteToDraw` (`viewport.cpp:664`) with a correct 3D
bounding box, and the existing sorter handles occlusion against terrain and buildings for
free. **Verify this against slopes and tall buildings before proceeding** — it was spike
exit criterion #1 for a reason.

### 4.3 Movement

Follow `disaster_vehicle.cpp`: integrate `x_pos`/`y_pos` in world units (`TILE_SIZE = 16`
units/tile), derive `z_pos` from `GetSlopePixelZ()` (`landscape.cpp:313`), update `tile`
on crossing, update `direction` from the movement delta, and refresh `coord` + the viewport
hash after every move.

**Integer-only.** `speed` and `progress` are fixed-point (`progress` is /256 of a tile unit,
matching upstream's convention). No floats anywhere in this path — see
[determinism rules](03-game-logic-guidelines.md#1-determinism-is-non-negotiable).

### 4.4 Tick scheduling with level-of-detail

You cannot fully tick 2,000 agents every tick at 30 Hz. Adopt upstream's amortisation
instinct (`CallVehicleTicks`, `RunTileLoop`) and add explicit LOD:

| Tier | Cadence | What runs |
|---|---|---|
| Movement | every tick | position integration for agents actually moving |
| Task step | every 4 ticks | task progress, arrival checks, state transitions |
| Needs | every 64 ticks | hunger/warmth/rest/happiness decay |
| Social/family | daily | relationships, pregnancy, aging |

Shard by `VillagerID % N` so each tick handles a fixed slice — bounded per-tick cost, no
spikes. Never iterate all villagers in a single tick for anything but the cheapest pass.

### Gate

2,000 villagers spawn, render correctly at all zooms, wander with placeholder movement, save
and load with stable IDs, and hold the tick budget. Commit as `v0.4-actors`.

---

## Phase 5 — Pathfinding

The biggest single engineering risk. Nothing in OpenTTD helps: YAPF is `Trackdir`-based and
structurally inapplicable.

### 5.1 Layered design

Don't build one pathfinder; build three, used at different scales.

| Layer | Technique | Use |
|---|---|---|
| **L1 Local** | Direct line-of-sight / greedy step | Same tile or adjacent; ~80% of queries. Never invoke A* for these. |
| **L2 Regional** | Grid A* with jump-point search over a walkability bitmap | Typical villager trips, ≤ ~64 tiles |
| **L3 Global** | Hierarchical: connected-component regions (16×16 blocks) + portal graph; A* over portals, L2 within | Long trips, and instant "is this reachable at all?" answers |

The L3 region graph is what makes "no reachable tree" cheap to answer — a common query that
naive pathfinders answer by exhaustively searching the whole map.

### 5.2 Flow fields for convergent traffic

When many villagers head to one destination (a stockpile, a construction site), compute one
Dijkstra flow field from the destination and let all of them follow the gradient. This turns
N path queries into 1 and is the standard fix for the "everyone hauls to the granary"
pattern that dominates this genre.

Cache flow fields per active destination; invalidate on traversability change.

### 5.3 Budget and amortisation

- **Hard cap the pathfinding budget per tick** (e.g. 1 ms). Queue overflow to the next tick;
  villagers wait a beat rather than the frame hitching. Non-negotiable: an unbounded
  pathfinder is an unshippable game.
- Cache paths with generation-counted invalidation driven by
  `NotifyTileTraversabilityChanged` (Phase 3.5).
- Reuse the open/closed set allocations across queries — `nodelist.hpp` in YAPF shows the
  pattern even though its node types don't apply.

### 5.4 Threading — carefully

Pathfinding is the one place worth parallelising. Rules:

- Workers read an **immutable snapshot** of the walkability bitmap, published once per tick.
- Results are collected in a deterministic order (sort by `VillagerID`) before being applied.
- No worker touches game state. Ever.

If you cannot guarantee both, stay single-threaded. Non-deterministic pathfinding destroys
replay testing (Phase 13) and produces bugs you cannot reproduce.

### 5.5 Local avoidance

Villagers should not stack or jitter. Keep it cheap: soft radial separation, right-of-way by
`VillagerID` on head-on conflict, `Path` tiles cheaper so traffic self-organises onto trails.
**Resist full crowd steering (RVO/ORCA)** — for this genre and agent count it costs more
than it returns.

### Gate

2,000 agents pathing on a 512×512 map, worst case (all heading to one point across the map)
inside the 1 ms budget, deterministic across runs given identical input. Commit as
`v0.5-pathfinding`.

---

## Phase 6 — Amputation II: economy → resources

Now that the sim's shape is known, cut the money economy.

### 6.1 Neutralise money, don't excise it

`Money` and `CommandCost` are threaded through every command. Excising them is weeks of
mechanical churn for no gameplay gain.

- **Keep `CommandCost`** purely as an error/`StringID` carrier. Its cost dimension becomes
  vestigial.
- Retain a single implicit `Company` with `OWNER_TOWN`-style semantics. Don't fight `Owner`;
  collapse it.
- Delete: `economy.cpp` payment logic, `subsidy*`, `cargopacket*` (distance-based payment),
  `linkgraph/`, `company_gui` finances, `graph_gui`, `league*`, `goal*`, `story*`,
  `cargomonitor*`, loans, bankruptcy, inflation, `currency*`.
- Replace build "cost" with **material requirements**: a build command validates that the
  required resources are available and reserves them.

### 6.2 Resource model

```cpp
enum class ResourceType : uint8_t {
	Wood, Stone, Flint, Hide, Bone, Sinew, Fibre, Clay,
	Berries, Roots, Fish, Meat, Grain, Herbs,
	Firewood, Tools, Clothing, Baskets, Pottery, ...
};

struct ResourceStack { ResourceType type; uint16_t amount; };
```

Design notes that matter:
- **Discrete integer amounts.** No fractional resources anywhere.
- Perishables need a spoilage clock. Decide now whether spoilage is per-stack (simple,
  cheap) or per-batch (realistic, expensive). **Recommend per-stack** with an average-age
  field.
- Resources have `weight` (hauling capacity) and `storage_class` (which buildings accept
  them).

### 6.3 `Storage` pool

```cpp
struct Storage : StoragePool::PoolItem<&_storage_pool> {
	TileArea location;
	StorageClasses accepts;              // bitset of storage classes
	uint16_t capacity_total;
	std::vector<ResourceStack> contents;
	uint16_t reserved[NUM_RESOURCE_TYPES];   // claimed by in-flight tasks
	bool     accept_enabled[NUM_RESOURCE_TYPES];  // player toggle
};
```

**The `reserved` field is essential and easy to forget.** Without it, ten villagers all
claim the same 5 units of wood, nine arrive to find nothing, and the job system thrashes.
Reserve on task assignment, release on completion *or* cancellation.

Register storages in a `kdtree` (`core/kdtree.hpp`) for "nearest storage accepting X" —
this is one of the highest-frequency queries in the game.

### 6.4 Settlement ledger

A cached aggregate of all storage contents, maintained incrementally (never recomputed by
scanning), for UI display and for the job system's "do we need more firewood?" decisions.

### Gate

No money in the UI or simulation. Resources are gathered, stored, reserved, and consumed.
Ledger matches a debug full recount. Commit as `v0.6-resources`.

---

## Phase 7 — Work & jobs

**This is the game.** Budget the most time and expect the most iteration.

### 7.1 Task model

```cpp
enum class TaskKind : uint8_t {
	Gather, Chop, Quarry, Hunt, Fish, Farm, Haul, Build, Craft, Tend, Idle, ...
};

struct Task : TaskPool::PoolItem<&_task_pool> {
	TaskKind kind;
	uint8_t  priority;
	TileIndex target;
	BuildingID building;
	ResourceType resource; uint16_t amount;
	VillagerID assignee;         // Invalid = unclaimed
	uint16_t progress;
	uint8_t  required_skill; TechID required_tech;
};
```

### 7.2 The scheduler

Per-tick, on a bounded slice:

1. **Generate** — producers post tasks: buildings needing input, construction sites needing
   materials, storages needing balancing, player-designated harvest areas.
2. **Score** — for each idle villager × each nearby candidate task:
   `score = priority × skill_match ÷ (travel_cost + 1)`.
3. **Assign** — greedy best-first, capped per tick. Reserve resources and target tiles
   atomically at assignment.
4. **Execute** — assignee walks, works, deposits.
5. **Release** — on completion *or* interruption *or* target invalidation, release all
   reservations.

### 7.3 Failure modes to design against from day one

Every game in this genre ships with some of these. Handle them structurally, not with
patches:

| Failure | Structural fix |
|---|---|
| Task thrashing (villagers re-decide every tick) | Commit to a task for a minimum duration; hysteresis on reassignment |
| Starvation (a low-priority task never runs) | Age-based priority boost |
| Reservation leaks (resources locked forever) | Reservations carry the owning `TaskID`; a periodic sweep releases orphans |
| Convergence (everyone hauls to the same place) | Flow fields (Phase 5.2) + per-destination assignment caps |
| Deadlock (A needs B needs A) | Cycle detection in the requirement graph; break by priority |
| Unreachable targets | L3 reachability check *before* assignment, not after the walk fails |

### 7.4 Player intent layer

Players express intent, not orders: harvest-area designations, storage priorities,
per-building worker counts, profession assignment, global production priorities. The
scheduler translates intent into tasks. Keep this separation clean — it's what makes the
game feel like Banished rather than an RTS.

### Gate

Full loop works unattended: designate a forest → villagers chop → haul to stockpile →
construction site consumes wood → building completes → workers staff it → it produces. No
deadlocks or leaks over a 4-hour headless run. Commit as `v0.7-jobs`.

---

## Phase 8 — Buildings & production

### 8.1 Data-driven building specs

Model on `ObjectSpec` (`newgrf_object.h:57`) — footprint size, height, views,
availability gating — but load from data files, not GRF:

```json
{
  "id": "longhouse",
  "size": [3, 2],
  "sprites": { "n": "building/longhouse/n", "e": "building/longhouse/e" },
  "requires_tech": "post_and_beam",
  "build_materials": [ {"wood": 40}, {"hide": 8}, {"fibre": 12} ],
  "build_labour": 240,
  "worker_slots": 0,
  "housing": 5,
  "warmth": 40,
  "storage": { "accepts": ["food"], "capacity": 100 },
  "terrain": { "max_slope": 1, "requires": ["Clear"] }
}
```

Hot-reload these in debug builds. Content iteration speed is the whole reason to be
data-driven.

### 8.2 Construction sites

`TileType::Construction` with delivered-materials tracking and a labour-progress counter.
Sites post `Haul` tasks for missing materials and `Build` tasks for labour. On completion,
convert to `TileType::Building` and register in the `Building` pool.

### 8.3 Production

Recipe-driven, replacing `Industry`'s rate-multiplier model:

```
inputs + worker-time + tech + season → outputs
```

Production rate scales with staffed workers, their skill, tool quality, and input
availability. Buildings pull inputs via `Haul` tasks and push outputs to storage.

Keep `Industry`'s `HistoryData` pattern (`industry.h:65-103`) — 24 months of rolling
production history is exactly what your economy graphs want, and it's already written.

### Gate

10+ building types placeable, buildable, staffable, productive, all from data files.
Commit as `v0.8-buildings`.

---

## Phase 9 — Population lifecycle

- **Needs**: hunger, warmth, rest, health, happiness. Decay on the 64-tick cadence; unmet
  needs degrade health then cause death. Warmth couples to season and firewood.
- **Families**: `Family` pool, pair formation, housing requirement, pregnancy, child→adult
  transition, inheritance of home.
- **Aging**: child / adult / elder, with age-dependent work capacity.
- **Skills**: per-profession experience with diminishing returns; schooling as an
  alternative to child labour (a real Banished-style tradeoff).
- **Death**: starvation, cold, age, accident, disease. Population collapse must be
  *recoverable but punishing*.

**Balance rule:** the population curve is the game's difficulty curve. Make every constant
here a hot-reloadable data value, not a literal in code. You will tune these hundreds of
times.

### Gate

A settlement survives 100 in-game years unattended under a scripted policy, or dies for
legible reasons. Commit as `v0.9-population`.

---

## Phase 10 — Technology tree integration

Your tech tree is being designed separately, so **the integration contract is what matters
here.** Define it before the tree is finished so both sides can proceed in parallel.

### 10.1 Data format

```json
{
  "id": "leather_tanning",
  "name": "STR_TECH_LEATHER_TANNING",
  "requires": ["stone_tools", "fire_control"],
  "research_cost": 120,
  "research_inputs": { "hide": 5 },
  "unlocks": {
    "buildings": ["tannery"],
    "recipes":   ["hide_to_leather"],
    "actions":   ["tan"],
    "modifiers": [ { "target": "clothing.warmth", "mul": 1.25 } ]
  }
}
```

### 10.2 The engine-side contract

The engine must not know your tech tree's shape. It needs exactly four things:

```cpp
bool IsTechUnlocked(TechID);
bool IsBuildingAvailable(BuildingType);        // = all required techs unlocked
bool IsRecipeAvailable(RecipeID);
int  GetTechModifier(ModifierKey, int base);   // multiplicative/additive stacking
```

Gate every unlockable thing through these. Model the pattern on `Engine::IsAvailable()` /
`ObjectSpec::IsAvailable()` — substitute "tech unlocked" for "introduction date" and the
plumbing into build menus and validation is already shaped correctly.

### 10.3 Progression mechanism

Decide explicitly and early, because it changes the whole feel:

| Mechanism | Feel | Fit |
|---|---|---|
| Accumulated "knowledge" from work (learning by doing) | Organic, Banished-like | **Recommended** for Lithic — knapping flint teaches you knapping |
| Dedicated researcher villagers at a building | Civ-like, explicit player control | Anachronistic for the setting, but very readable |
| Milestone unlocks (population/production thresholds) | Automatic, low-friction | Good as a supplementary layer |

A hybrid works: learning-by-doing accumulates knowledge, a shaman's hut converts knowledge
into completed techs.

**Circular-dependency validation at load time, with a clear error.** A tech graph with a
cycle should fail to load loudly, not deadlock the game silently.

### 10.4 UI

Tech tree screen with prerequisite graph rendering. `widget.cpp`'s auto-layout will not do
graph layout for you — plan a custom-drawn window with hand-authored tier/column positions
in the data file (much simpler and prettier than automatic graph layout).

### Gate

Tech data loads and validates, unlocks gate buildings/recipes/actions, research progresses,
tree screen renders. Commit as `v0.10-tech`.

---

## Phase 11 — UI layer

All new windows on the existing framework. Follow the declarative `NWidgetPart` tree pattern
from any surviving `*_gui.cpp`.

Core screens: settlement overview (population, resources, alerts), villager list with
sort/filter (reuse `GUIList`/`sortlist.h` — it's excellent), villager inspector, building
inspector with staffing, build menu grouped by tech tier, storage manager, profession
allocation, tech tree, production graphs (reuse the `HistoryData` pattern from
`industry.h`), alerts/notifications (`news_gui` is repurposable).

**Do not fight the widget framework.** It is idiosyncratic and mature. Learn
`NWidgetPart`/`NWID_*`/`SetFill`/`SetResize` properly once, and use `_gui.cpp` files as
templates.

Keep all display strings in `lang/` from the first window — retrofitting localisation is
miserable, and `strgen` is already set up.

---

## Phase 12 — World generation & scenario

Retune `tgp.cpp` for small, dense, hand-feeling maps rather than large transport maps.
Post-generation passes seed resources: forest species by altitude/moisture, rock and flint
outcrops, forage patches, fish in water, soil fertility gradients, starting-area validation
(guaranteed water + wood + buildable flat land within a radius).

Keep `heightmap.cpp` — hand-authored heightmaps make good curated scenarios.

Add a scenario definition: starting villagers, starting resources, starting techs, season,
difficulty modifiers.

---

## Phase 13 — Save/load consolidation & the replay harness

Adapt `saveload/` chunks to your pools. Delete the pre-fork compatibility layer
(`saveload/compat/`, `oldloader*`) — you have no legacy saves. Establish version-bump
discipline (see [guidelines §6](03-game-logic-guidelines.md#6-saveload-discipline)).

**Build the replay harness here — it's the highest-value test infrastructure you'll have.**
Because every mutation is a command, you can:

1. Record `(tick, command, args)` to a log.
2. Replay from a seeded new game.
3. Assert a hash of full game state matches at checkpoints.

This catches determinism regressions, save/load asymmetry, and subtle sim bugs that no unit
test will find. Run it in CI over several recorded multi-hour sessions.

**Go one step further than a hash — add per-field state diffing [from OpenRCT2].** OpenRCT2
ships this and it's the piece that makes replay testing *usable*: `ReplayManager.cpp` (890 LOC)
records and replays, `GameStateSnapshots.cpp` (801 LOC) compares two snapshots **field by
field** (`CompareSpriteDataPeep`, `CompareSpriteDataCommon`, …) and reports which member of
which entity diverged, and `EntitiesChecksum` is a SHA-1 over all entity state.

The difference in practice: a bare hash tells you *"state diverged at tick 91,338."* Field
diffing tells you *"`Villager::hunger` differs on villager 4,102 at tick 91,338: 41 vs 42."*
The first sends you bisecting for a day; the second names the bug. Given that you're writing the
harness anyway, write the diff generator alongside it — a `SL_`-style field table per pool gets
you comparison and serialisation from one declaration.

Also copy OpenRCT2's practice of keeping a **versioned corpus of recorded replays** as a
separate asset (their `assets.json` pins a `replays` archive by SHA-256) rather than committing
large binaries into the source tree.

---

## Phase 14 — Modding surface (optional)

`3rdparty/squirrel` survives Phase 1. Expose a small API — read game state, register
content, hook events — and you have modding without writing a VM. Note the tradeoff:
scripted content that mutates state must go through commands to stay deterministic and
replay-safe.

---

## Risk register

| Risk | Impact | Mitigation |
|---|---|---|
| GPL v2 blocks the business model | **Fatal** | Resolve in Phase 0 before any code |
| Pathfinding doesn't scale | **Severe** | Spike measures it in Phase 0; layered design + hard tick budget |
| Job system deadlocks/thrashes | **Severe** | Design failure modes in from the start (§7.3); headless soak tests |
| Sprite sorter misbehaves with your art | High | Verify in Phase 0 spike before commissioning art |
| GRF pipeline poisons art workflow | High | Phase 2 before any art production |
| Tile representation chosen wrong | **Severe** | Decide Option A vs B in Phase 3 §3.1 *before* Phase 8; retrofitting element lists after buildings exist reworks every building |
| Tile layout churn | Medium | Change once, deliberately, in Phase 3; maintain `docs/tile-layout.md` |
| Money/`Owner` excision rabbit hole | Medium | Neutralise, don't excise (§6.1) |
| NewGRF removal is tangled | Medium | Delete per-feature with the owning subsystem; stub callbacks first |
| Upstream divergence | Low | Keep `upstream` remote; minimise edits to `viewport.cpp`/`blitter/`/`window.cpp` |
| Scope creep in a genre that invites it | High | Vertical slice first; feature-freeze for a playable loop before adding systems |

---

## Solo-developer sequencing note

The phase table sums to a multi-year effort at one FTE. If that's your situation, this is the
order that keeps a playable thing in your hands throughout:

**Phases 0 → 1 → 2 → 3 → 4 → 5 → 6 → 7**, then stop and play it. Phases 0–7 produce a game
where villagers gather, haul, and build — the recognisable core. Phases 8–11 make it a
*game* worth playing for hours. Phases 12–14 make it worth replaying.

If you must cut, cut Phase 14 first, then trim Phase 12 to hand-authored heightmaps only.
Never cut Phase 13's replay harness — it pays for itself within weeks.
