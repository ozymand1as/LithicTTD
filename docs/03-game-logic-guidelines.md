# Game logic implementation guidelines

Rules for writing simulation code in a Lithic settlement game built on an OpenTTD fork.

These are the conventions that keep a simulation of thousands of agents deterministic,
debuggable, performant, and save-safe. Most of them are cheap to follow from day one and
expensive to retrofit. Where OpenTTD already established a convention, follow it — you gain
consistency with 130k lines of surviving code and you keep upstream cherry-picks viable.

---

## Contents

1. [Determinism is non-negotiable](#1-determinism-is-non-negotiable)
2. [All mutation goes through commands](#2-all-mutation-goes-through-commands)
3. [Amortise everything; never iterate everything](#3-amortise-everything-never-iterate-everything)
4. [Performance budgets](#4-performance-budgets)
5. [Simulation and presentation are separate](#5-simulation-and-presentation-are-separate)
6. [Save/load discipline](#6-saveload-discipline)
7. [Technology gating](#7-technology-gating)
8. [Content is data, not code](#8-content-is-data-not-code)
9. [Reservations and resource integrity](#9-reservations-and-resource-integrity)
10. [Spatial queries](#10-spatial-queries)
11. [Agent AI structure](#11-agent-ai-structure)
12. [Balance and tuning](#12-balance-and-tuning)
13. [Testing](#13-testing)
14. [Code conventions](#14-code-conventions)
15. [Anti-patterns](#15-anti-patterns)

---

## 1. Determinism is non-negotiable

Even in single-player. Determinism is what makes replay testing (§13) possible, makes bug
reports reproducible from a save + command log, and keeps a future multiplayer or
cloud-save-verification option open. Once you lose it, you cannot get it back cheaply.

### Rules

**No floating point in simulation code.** Ever. Not for movement, not for production rates,
not for needs decay, not for probabilities. Floats differ across compilers, optimisation
levels, and CPUs.

```cpp
// WRONG
float speed = 1.5f;
pos += speed * delta;

// RIGHT — fixed-point, following upstream's convention
uint16_t speed;        // in world-units per tick << 8
uint8_t  progress;     // fractional accumulator, /256
uint32_t step = speed + progress;
pos      += step >> 8;
progress  = step & 0xFF;
```

Floats are fine in rendering, UI layout, and audio — anything downstream of the sim that
never feeds back into it.

**Use the game's seeded randomiser, never `rand()` or `std::mt19937` with arbitrary
seeding.**

```cpp
#include "core/random_func.hpp"

uint32_t r = Random();               // game-state randomiser, saved/restored
uint32_t n = RandomRange(10);        // 0..9
uint32_t i = InteractiveRandom();    // ONLY for cosmetics — NOT saved, NOT deterministic
```

`InteractiveRandom()` exists for things that must not perturb game state — tooltip flavour
text, idle animation variation. Using it in the sim is a determinism bug that will take days
to find. Using `Random()` in a render path is the same bug in reverse.

**No iteration over unordered containers in simulation code.** `std::unordered_map`
iteration order varies between runs and implementations. Use `std::map`, sorted vectors, or
pool iteration order.

```cpp
// WRONG — iteration order is not guaranteed
for (auto &[id, task] : this->unordered_task_map) { ... }

// RIGHT — pool iteration is index-ordered and stable
for (Task *t : Task::Iterate()) { ... }
```

**No pointer values in decisions.** Sorting or hashing by address, or using pointer
comparison for tie-breaking, is non-deterministic across runs. Tie-break on stable IDs:

```cpp
// WRONG
std::sort(candidates.begin(), candidates.end());                       // sorts by address
// RIGHT
std::sort(candidates.begin(), candidates.end(),
          [](const Villager *a, const Villager *b) { return a->index < b->index; });
```

**No wall-clock time or thread scheduling in the sim.** All timing comes from
`TimerGameTick::counter`. If work is parallelised (only pathfinding should be — see the
[rework plan §5.4](02-engine-rework-plan.md#54-threading--carefully)), results must be
collected and applied in a deterministic order.

**Beware unspecified evaluation order.** `f(Random(), Random())` has no guaranteed order of
evaluation. Sequence random draws into named locals:

```cpp
uint32_t a = Random();
uint32_t b = Random();
Spawn(a, b);
```

### Enforce it

Add a CI check that greps simulation directories for `float`, `double`, `rand()`,
`unordered_`, `std::chrono`, and `InteractiveRandom`. A grep-level guard catches 90% of
violations at review time, which is far cheaper than finding them via a replay divergence
three months later.

---

## 2. All mutation goes through commands

Follow the existing command architecture (`command_type.h`, `command_func.h`). Every player
action and every scripted/AI action is a registered command with a test pass and an execute
pass.

```cpp
CommandCost CmdBuildStructure(DoCommandFlags flags, TileIndex tile, BuildingType type)
{
	/* --- validation: must be side-effect free, and identical in both passes --- */
	if (!IsBuildingAvailable(type)) return CommandCost(STR_ERROR_TECH_NOT_RESEARCHED);

	const BuildingSpec *spec = BuildingSpec::Get(type);
	CommandCost ret = CheckBuildableArea(tile, spec->size);
	if (ret.Failed()) return ret;
	if (!HasMaterials(spec->build_materials)) return CommandCost(STR_ERROR_INSUFFICIENT_MATERIALS);

	/* --- execution: only under Execute --- */
	if (flags.Test(DoCommandFlag::Execute)) {
		ReserveMaterials(spec->build_materials);
		CreateConstructionSite(tile, type);
		MarkTileDirtyByTile(tile);
	}
	return CommandCost();
}
```

### Rules

- **The test pass must be a pure function of game state.** No mutation, no `Random()` calls
  whose results the execute pass depends on, no caching side effects. Violating this makes
  the UI's build-preview lie and breaks replay.
- **Test and execute must agree.** If the test pass succeeds, the execute pass must succeed.
  Divergence here produces half-applied state, the worst class of bug in this architecture.
- **Fail with a `StringID`, not a bool.** The player needs to know *why*. `CommandCost`
  carries this; use it.
- **Never mutate game state from UI code, from a `draw_tile_proc`, or from a rendering
  path.** The viewport can be redrawn any number of times per tick, including zero.
- **The internal simulation may bypass the command layer** for its own high-frequency
  bookkeeping (a villager stepping forward, a tile's growth tick) — commands are for
  discrete, player-or-script-initiated, undoable-in-principle actions. Don't route 2,000
  movement updates per tick through the command dispatcher.

---

## 3. Amortise everything; never iterate everything

The core performance discipline. OpenTTD survives 4096×4096 maps because nothing scans the
whole map per tick, and neither should you.

### The tile loop pattern

Reuse `RunTileLoop()` (`landscape.cpp:806`) as-is. Its Galois LFSR visits every tile exactly
once per `TILE_UPDATE_FREQUENCY` ticks at constant per-tick cost, in a deterministic
pseudorandom order. Put per-tile simulation in `tile_loop_proc` and derive sub-cadences from
the tile coordinate plus the tick counter, exactly as `TileLoop_Trees` does
(`tree_cmd.cpp:854`):

```cpp
uint32_t cycle = 11 * TileX(tile) + 9 * TileY(tile) + (TimerGameTick::counter >> 8);
if ((cycle & 15) != 15) return;   // this tile's slow work runs 1 visit in 16
```

The co-prime multipliers spread work evenly so you never get a whole map region updating on
the same tick.

### The agent LOD pattern

| Tier | Cadence | Content |
|---|---|---|
| Movement | every tick | position integration, moving agents only |
| Task step | every 4 ticks | progress, arrival, state transitions |
| Needs | every 64 ticks | hunger/warmth/rest/happiness decay |
| Social | daily | relationships, pregnancy, aging |
| Scheduler | every tick, bounded slice | assign at most N tasks per tick |

Shard by ID so each tick handles a fixed slice:

```cpp
/* Needs update: each villager every 64 ticks, 1/64 of them per tick. */
const uint slice = TimerGameTick::counter & 63;
for (Villager *v : Villager::Iterate()) {
	if ((v->index.base() & 63) != slice) continue;
	UpdateNeeds(v);
}
```

For large pools, maintain per-slice index vectors instead of iterating-and-skipping — at
2,000 villagers the skip is cheap, at 20,000 it isn't.

### Caches must be derived, incremental, and verifiable

Aggregates (settlement resource totals, population counts, building counts) are maintained
incrementally on change, never recomputed by scanning. Follow the existing pattern: caches
are `NOSAVE`, rebuilt on load (`RebuildTownCaches()` in `saveload/town_sl.cpp:29` is the
model), and validated in debug builds.

`cachecheck.cpp` is the pattern to copy: in debug builds, periodically recompute every cache
from scratch and assert it matches. This catches the entire class of "cache drifted from
reality" bugs, which are otherwise nearly impossible to find. **Write your `CheckCaches()`
early, not after the bugs appear.**

---

## 4. Performance budgets

Declare budgets, measure against them in CI, and treat a regression as a build failure.

Targets, at **30 sim ticks/s, 512×512 map, 2,000 villagers, 300 buildings**:

| Subsystem | Budget/tick | Notes |
|---|---:|---|
| Agent movement | 2.0 ms | Only actually-moving agents |
| Pathfinding | **1.0 ms hard cap** | Queue overflow to next tick; never exceed |
| Task scheduler | 1.0 ms | Bounded assignments/tick |
| Tile loop | 1.0 ms | Fixed cost by construction |
| Needs/lifecycle | 0.5 ms | 1/64 slice |
| Production | 0.5 ms | Building slice |
| Everything else | 1.0 ms | |
| **Total sim** | **≤ 7 ms** | Leaves ~26 ms for render at 30 fps |

Rules:

- **The pathfinding cap is hard.** Overflow queues to the next tick. A villager idling one
  extra tick is invisible; a 40 ms frame hitch is not. This is the single most common cause
  of unshippable performance in this genre.
- Use the built-in `framerate_gui.cpp` / `PerformanceMeasurer` / `PerformanceAccumulator`
  infrastructure — it already exists, already breaks down per-subsystem, and already has a
  UI. Add your own `PerformanceElement` entries rather than building new instrumentation.
- Benchmark in CI on a fixed saved game with a fixed tick count. Fail the build on
  >10% regression.
- Keep the hot structs small. `Villager` ≤ 128 bytes (§[rework plan 4.1](02-engine-rework-plan.md#41-the-villager-pool)).
  Put rare data in side tables keyed by ID.
- Profile before optimising, always. The bottleneck in this genre is almost always
  pathfinding or the task scheduler's candidate scoring — not the thing you assumed.

---

## 5. Simulation and presentation are separate

| Layer | May do | May **not** do |
|---|---|---|
| Simulation | mutate game state, use `Random()`, integers only | touch sprites, read viewport/zoom/scroll state, call `InteractiveRandom()` |
| Presentation | read game state, interpolate, use floats and `InteractiveRandom()` | mutate any game state |

The sim runs at a fixed tick rate; rendering runs at display rate and interpolates. Purely
visual state (animation frames, smoke, particles, sprite caches) is `NOSAVE` and rebuilt
freely — `MutableSpriteCache` in `vehicle_base.h` shows the convention.

The practical test: **if you paused the sim, would the render still be correct?** And: **if
you never rendered a frame, would the sim produce identical results?** Both must be yes. The
second is what makes headless soak testing and CI benchmarking possible.

---

## 6. Save/load discipline

`saveload/` is a mature chunked, versioned serialiser with declarative per-struct field
tables. Follow its conventions exactly.

- **One chunk per pool.** `VLGR` (villagers), `BLDG` (buildings), `STOR` (storages), `TASK`
  (tasks), `TECH` (research), `FMLY` (families).
- **Bump the save version for any field change**, and add a compat entry. There is no such
  thing as a harmless field addition once you have players.
- **Never save derived state.** Mark caches `NOSAVE` and rebuild in the post-load pass.
  Rebuilding is more code but it eliminates the entire class of "loaded a save with a stale
  cache" bugs.
- **Save/load must be symmetric and lossless.** The replay harness (§13) verifies this: save
  → load → hash must equal the pre-save hash.
- **Delete the pre-fork compat layer** (`saveload/compat/`, `oldloader*`). You have no
  legacy saves; carrying dead compatibility code is pure liability.
- **Version your data files too.** Building specs, recipes, and the tech tree are content; a
  save references them by string ID. Decide the policy explicitly: does loading an old save
  with a newer tech tree work? (Recommended: yes, by string ID, with unknown IDs logged and
  dropped with a warning — not a crash.)

---

## 7. Technology gating

The tech tree is designed separately, so the engine must not know its shape. It needs exactly
four entry points, and every unlockable thing routes through them:

```cpp
bool IsTechUnlocked(TechID tech);
bool IsBuildingAvailable(BuildingType type);   // all required techs unlocked
bool IsRecipeAvailable(RecipeID recipe);
int  GetTechModifier(ModifierKey key, int base);
```

### Rules

- **Never hardcode a tech check in gameplay code.** `if (tech == TECH_BOW_MAKING)` scattered
  through the codebase is how a tech tree becomes unchangeable. Gate on the *capability*, not
  the tech: `if (IsActionAvailable(ACTION_HUNT_RANGED))`.
- **Validate the graph at load time.** Cycles, missing prerequisites, and dangling unlock
  references must produce a clear load-time error, not a silent deadlock.
- Modifiers stack in a defined, documented order (additive within a category, then
  multiplicative across categories — pick one and write it down). Integer maths only: express
  multipliers as `×N/256` fixed-point, not floats.
- Gating pattern to follow: `Engine::IsAvailable()` and `ObjectSpec::IsAvailable()` show how
  "this buildable thing is not yet available" threads through the build UI, validation, and
  tooltips. Substitute tech state for introduction date.
- Unlocking must be **monotonic** — nothing ever becomes un-researched. Non-monotonic
  progression multiplies the state space and every UI has to handle it.

---

## 8. Content is data, not code

Buildings, recipes, resources, techs, professions, and tuning constants live in data files
(JSON via the vendored `3rdparty/nlohmann`), not in C++ tables.

```
data/
  buildings/    longhouse.json, tannery.json, drying_rack.json ...
  recipes/      hide_to_leather.json, flint_to_blade.json ...
  resources/    resources.json
  tech/         tech_tree.json
  balance/      population.json, needs.json, production.json
  gfx/          atlases + sprite manifests
  lang/         .txt string files (strgen)
```

Rules:

- **Hot-reload in debug builds.** Content iteration speed is the entire justification for
  being data-driven. If you can't reload a building spec without restarting, you've paid the
  cost and skipped the benefit.
- **Validate on load, with actionable errors.** `"longhouse.json: unknown resource 'leathr'
  in build_materials — did you mean 'leather'?"` Do not let bad content reach the sim.
- **String IDs in data, numeric indices in the sim.** Resolve once at load; game code uses
  indices for cache locality.
- **All display text goes through `strgen`/`lang/`** from the very first window. Retrofitting
  localisation is miserable, and the pipeline already exists.
- **No magic numbers in C++.** Every balance constant is a named entry in `data/balance/`.
  You will tune these hundreds of times, and each one hardcoded is a recompile.

---

## 9. Reservations and resource integrity

The most common source of subtle bugs in this genre. Resources and target tiles are claimed
by in-flight tasks, and every claim must be released on every exit path.

```cpp
struct Storage {
	std::vector<ResourceStack> contents;
	uint16_t reserved[NUM_RESOURCE_TYPES];   // claimed but not yet collected

	uint16_t Available(ResourceType t) const { return Get(t) - this->reserved[t]; }
};
```

### Rules

- **Reserve at assignment, not at arrival.** Otherwise ten villagers walk to the same 5 units
  of wood and nine walk back empty.
- **Every reservation carries its owning `TaskID`.** Anonymous reservations cannot be
  audited or cleaned up.
- **Release on completion, cancellation, *and* agent death.** Enumerate the exit paths; there
  are more than you think. Prefer an RAII-style or task-destructor-driven release over
  scattered manual `Release()` calls.
- **Sweep for orphans.** A periodic pass verifies every reservation has a live owning task,
  releasing any that don't, and logs a warning in debug. This turns "resources mysteriously
  locked forever after 6 hours" into a caught bug.
- **Assert conservation in debug.** Total resources in the world = storage + carried +
  in-production + reserved. Any drift is a bug; catch it at the tick it happens, not in a
  player's 40-hour save.

The same discipline applies to tile claims: a tree being chopped is claimed so two
woodcutters don't target it. `TileSim::claim` (§[rework plan 3.1](02-engine-rework-plan.md#31-widen-the-tile-struct--deliberately-once))
holds this.

---

## 10. Spatial queries

"Nearest X" is the highest-frequency query class in the game. Never linear-scan.

- **`core/kdtree.hpp`** — already used for towns/stations, has `FindNearest` and
  `FindContained`. Use for sparse, mostly-static entity sets: buildings, storages, resource
  patches. Note it's a static-ish structure; rebuild on bulk change rather than churning it
  per-insertion.
- **Bucketed grids** for dense dynamic sets: villagers. Copy the viewport hash pattern
  (`_vehicle_viewport_hash`, `vehicle.cpp`).
- **Region graph** (from the [pathfinder's L3 layer](02-engine-rework-plan.md#51-layered-design))
  answers *reachability* before you spend a query on *distance*. "Nearest tree" is worthless
  if it's across a lake.

**The ordering rule: reachable first, then nearest.** Answering these in the wrong order is
the classic bug where villagers repeatedly walk toward something they can never reach.

And: **the nearest resource is not always the right one.** Score candidates on
`travel_cost + scarcity + player_priority`, not raw distance alone.

---

## 11. Agent AI structure

Keep villager AI a flat, inspectable state machine. Resist hierarchical behaviour trees and
utility-AI frameworks — with 2,000 agents on a tick budget, you need cheap and debuggable
more than you need expressive.

```cpp
enum class VillagerState : uint8_t {
	Idle, MovingToTask, Working, Hauling, Depositing,
	Eating, Sleeping, SeekingWarmth, Fleeing, Dying,
};
```

### Rules

- **One state, one tick step, one exit condition.** No nested state machines.
- **Commit to a task for a minimum duration.** Re-deciding every tick produces the classic
  visual bug where villagers oscillate between two targets forever. Add hysteresis: a new
  task must beat the current one by a margin, not merely tie.
- **Interruption is explicit and enumerated.** Starving, freezing, danger, and player
  reassignment interrupt work — nothing else does. An open-ended interruption system becomes
  unreasonable to debug.
- **Every state must have a timeout** with a defined fallback to `Idle`. A stuck agent must
  self-recover rather than freeze forever; log it in debug so you find the underlying cause.
- **Needs pressure drives priority, not state.** A hungry villager doesn't get a special
  state machine; hunger raises the score of food-related tasks. This keeps one decision
  system rather than several competing ones.
- **Make it inspectable.** A debug overlay showing each villager's state, current task,
  target, and path is the single highest-value debugging tool you will build. Build it in
  Phase 4, not Phase 11.

---

## 12. Balance and tuning

- **Every balance number lives in `data/balance/`.** No exceptions.
- **Define the reference scenario**: 5 starting villagers, temperate map, standard difficulty.
  All balance claims are relative to it, so "is this too hard?" has a shared meaning.
- **Instrument the economy.** Log population, food stores, resource throughput, and death
  causes per in-game year to CSV. Balance from the graphs, not from vibes.
- **Automate soak runs.** A headless run with a scripted policy over 100 in-game years, in
  CI, catches balance regressions and long-horizon bugs (reservation leaks, cache drift,
  population death spirals) that no manual playtest will.
- **Population is the difficulty curve.** In this genre the failure mode is always the same:
  either the population grows without limit (no tension) or dies from an unreadable cause
  (unfair). Tune for a curve that grows when managed and shrinks when neglected, with a
  *legible* cause of death.
- **Reuse the `HistoryData` pattern** (`industry.h:65-103`) — 24 months of rolling history is
  already implemented and is exactly what both your UI graphs and your balance CSVs want.

---

## 13. Testing

| Layer | Tool | What |
|---|---|---|
| Unit | the existing Catch2 setup (`src/tests/`, `3rdparty/catch2`) | Pure functions: pathfinding cost, recipe resolution, tech graph validation, fixed-point maths |
| Content | Load-time validators in CI | Every data file parses, references resolve, tech graph is acyclic |
| **Replay** | Command-log record/replay | **The highest-value test you have** |
| Save round-trip | Property test | save → load → hash == pre-save hash |
| Soak | Headless multi-hour runs | Deadlocks, leaks, cache drift, balance |
| Performance | Fixed save + fixed tick count in CI | Budget regression (§4) |

### The replay harness

Because every player action is a command, you can record `(tick, command, args)` and replay
it from a seeded new game, hashing the full game state at checkpoints.

This one harness catches: determinism regressions, save/load asymmetry, cache drift,
uninitialised state, and accidental float or `unordered_` introduction. It is worth more than
every unit test you will write, and it costs a few days.

Keep a library of recorded sessions in CI: an early-game 30 minutes, a mid-game hour, and a
large late-game settlement. When the hash diverges, bisect on the tick.

Note the corollary for `src/tests/`: the suite already builds and passes (94 cases,
2,145 assertions). Keep it green through the amputation phases rather than letting it rot —
it's free regression coverage on `core/` and the tile maths.

---

## 14. Code conventions

Follow `CODINGSTYLE.md` from upstream. Consistency with 130k surviving lines is worth more
than your personal preference, and it keeps upstream cherry-picks clean.

- Tabs for indentation, K&R-ish braces, `/** Doxygen */` on every non-trivial function.
- `_leading_underscore` for globals, `CamelCase` functions, `snake_case` members.
- `enum class` with `EnumBitSet` for flags — do not hand-roll bitfields.
- Strong typedefs for IDs (`PoolID`, `StrongType::Typedef`). This is not ceremony: it
  prevents the entire class of "passed a `BuildingID` where a `TileIndex` was expected" bug,
  which is otherwise endemic in code like this.
- New subsystems get their own files; **avoid editing `viewport.cpp`, `blitter/`,
  `window.cpp`, and `widget.cpp`** so upstream fixes stay cherry-pickable.
- Update `docs/tile-layout.md` in the same commit as any tile-layout change. Non-negotiable —
  this is the discipline whose absence upstream makes OpenTTD's tile layer hard to modify.
- Assert liberally; the codebase is built with `OPTION_USE_ASSERTS` on by default in debug and
  the asserts are load-bearing documentation.

---

## 15. Anti-patterns

Things that will hurt, drawn from what this genre and this codebase get wrong:

| Anti-pattern | Why it hurts |
|---|---|
| Unbounded pathfinding | Frame hitches; the #1 cause of "unshippable" in this genre |
| Full-map scans per tick | Kills scaling; the tile loop exists precisely to avoid this |
| Floats in the sim | Silently destroys determinism; the bug surfaces months later |
| Re-deciding tasks every tick | Villagers oscillate; looks broken even when logic is correct |
| Reserving on arrival instead of assignment | Nine of ten villagers arrive to nothing |
| Anonymous reservations | Unauditable resource leaks |
| Saving derived state | Stale-cache bugs on load, forever |
| Hardcoded tech checks | Makes the tech tree unchangeable — the one thing being designed in parallel |
| Magic numbers in C++ | Every balance tweak becomes a recompile |
| Mutating state in draw code | The viewport redraws an arbitrary number of times per tick |
| Fighting the widget framework | It's idiosyncratic but mature; learn it once |
| Excising `Money`/`Owner` | Weeks of churn for zero gameplay gain — neutralise instead |
| Reformatting upstream-shared files | Forfeits years of free renderer and GUI bugfixes |
| Building the UI before the sim | You'll build the wrong UI |
| Hierarchical behaviour trees for 2,000 agents | Costs more than a flat state machine returns |
| Full crowd steering (RVO/ORCA) | Same — cheap separation is enough for this genre |
| Skipping the replay harness | The one piece of test infrastructure that pays for itself in weeks |

---

## Quick reference: files worth reading before you start

| File | Why |
|---|---|
| `src/landscape.cpp:806` `RunTileLoop` | The amortisation pattern you'll use everywhere |
| `src/tree_cmd.cpp:836` `TileLoop_Trees` | A working growth/spread simulation to adapt |
| `src/clear_cmd.cpp:284` `TileLoop_Clear` | Ground state and field ageing |
| `src/tile_cmd.h` + `src/landscape.cpp:69` | The tile-type dispatch table — your main seam |
| `src/viewport.cpp:1211` `ViewportAddLandscape` | How tiles become sprites |
| `src/viewport_sprite_sorter.h:16` | The bounding-box sort that makes the whole look work |
| `src/vehicle.cpp:1149` `ViewportAddVehicles` | Actor rendering via spatial hash — port this |
| `src/disaster_vehicle.cpp` | Free movement in world coords, ignoring tracks |
| `src/core/pool_type.hpp` | Entity storage for all your new types |
| `src/core/kdtree.hpp` | Nearest-neighbour queries |
| `src/object_cmd.cpp` | Best template for placeable multi-tile structures |
| `src/industry.h:64` | Production + `HistoryData` rolling history |
| `src/cachecheck.cpp` | The cache-validation pattern — copy it early |
| `src/saveload/town_sl.cpp:29` | Post-load cache rebuild convention |
| `CODINGSTYLE.md`, `docs/landscape.html` | House style, and the tile bit-layout you're replacing |
