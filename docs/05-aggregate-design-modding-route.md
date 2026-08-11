# The aggregate design as a NewGRF + GameScript total conversion

**The question:** drop individual people. Model the settlement as aggregates (population,
resources, professions, known technologies). Gameplay is materials gathering + transport +
numeric settler job assignment. Caravans and hunting parties behave like TTD vehicles. **Can
this be done as a graphics overhaul plus a bunch of game scripts, with no C++ fork?**

**Verdict: yes — substantially. This is the design OpenTTD's modding surface was actually
built for**, and the pivot removes essentially every hard blocker identified in
[the OpenTTD assessment](01-feasibility.md) and [the OpenRCT2 assessment](04-openrct2-feasibility.md).

Effort drops from *years of C++ archaeology* to *months of art, NML, and Squirrel*. Several
mechanics you'd have written by hand — settlement growth driven by food delivery, technology
gating of caravan types, stockpile management, aggregate production levels — turn out to be
**native engine features you configure rather than build**.

There is one significant limitation and it is not negotiable from the mod layer: **you cannot
create custom GUI windows.** Everything else is either native, or a workable contortion. That
limitation is what drives the recommendation in [§8](#8-recommendation-mod-first-then-a-thin-patch).

---

## Contents

- [1. Why the pivot changes everything](#1-why-the-pivot-changes-everything)
- [2. Licensing — the sleeper advantage](#2-licensing--the-sleeper-advantage)
- [3. What maps natively](#3-what-maps-natively)
- [4. The architecture](#4-the-architecture)
- [5. The hard limits](#5-the-hard-limits)
- [6. Workable contortions](#6-workable-contortions)
- [7. Build plan](#7-build-plan)
- [8. Recommendation](#8-recommendation-mod-first-then-a-thin-patch)

---

## 1. Why the pivot changes everything

The previous assessments identified four things that made the Banished-style design expensive.
The aggregate design deletes all four:

| Blocker from the fork plan | Cost there | Status under the aggregate design |
|---|---|---|
| Agent pathfinding (nothing reusable, 6–10 wk, permanent hotspot) | Severe | **Gone.** Caravans are vehicles; YAPF already pathfinds them on roads. |
| Job/labour system (8–12 wk, "this is the game") | Severe | **Reduced to a number.** `SetProductionLevel` on an industry. |
| Individual needs/families/lifecycle | 4–6 wk | **Gone.** Town population + food delivery, both native. |
| Tile model rework (widen struct / element lists) | 3–4 wk + risk | **Gone.** You use OpenTTD's tiles as they are. |

And critically: **you stop deleting 167k lines of C++ and stop maintaining a fork.** You
consume upstream OpenTTD as a shipped binary. Renderer fixes, platform ports, and performance
work arrive for free forever.

---

## 2. Licensing — the sleeper advantage

This is worth flagging first because it inverts the biggest strategic problem with the fork.

If you **do not modify OpenTTD's C++**, your deliverables are:

- a **NewGRF** — a data file the engine interprets
- a **GameScript** — Squirrel source the engine's embedded VM interprets
- art, sound, strings, a scenario

None of these are linked into the GPL v2 binary. They are data and scripts consumed by an
unmodified interpreter — the same category as a Doom WAD or a game's Lua mod. **The widely
accepted reading is that they are separate works and may be licensed however you like,
including proprietary.**

Two honest caveats:

1. **Get actual counsel before betting a business on it.** The FSF takes a narrower view of
   plugins that share complex data structures with the host than of ones calling a documented
   API. A GameScript calling `ScriptIndustry::SetProductionLevel()` is squarely in the
   documented-API category, but "widely accepted" is not "adjudicated."
2. **BaNaNaS** — OpenTTD's in-game content service — requires GPL-compatible licensing for
   uploads. If you want in-game discovery you accept that; if you distribute independently
   (Steam, itch, your own installer bundling unmodified OpenTTD) you don't.

Either way, **you can ship OpenTTD's binary unmodified alongside your content** and satisfy
GPL v2 by pointing at OpenTTD's already-public source. No source obligation on your own work.

Compare with the fork, where your entire game becomes GPL v2 and console ports are effectively
blocked. This alone may be the strongest argument for the mod route.

---

## 3. What maps natively

Every row below was verified against the source, not assumed.

| Your design element | Engine mechanism | Status |
|---|---|---|
| **Settlement as an aggregate** | `Town` — `GetPopulation()`, `GetHouseCount()`, `SetGrowthRate()`, `ExpandTown()`, `FoundTown()` | **Native** |
| **Settlement grows when fed** | Cargo property `town_acceptance_effect` (`cargotype.h:89`). Define a `FOOD` cargo with `TAE_FOOD`; delivering it to the settlement drives growth. | **Native — configure, don't build** |
| **Displaying aggregate state** | `ScriptTown::SetText()` puts arbitrary text in the town window; `SetCargoGoal()` shows per-cargo requirements | **Native** |
| **Resources** | Cargo types — **64 max** (`NUM_CARGO`, `cargo_type.h:75`). Your ~20 (wood, flint, hide, sinew, fibre, clay, berries, fish, meat, grain, herbs, firewood, tools, clothing, baskets, pottery…) fit easily. NewGRF sets `weight`, `initial_payment`, `transit_periods`, capacity `multiplier` per cargo. | **Native** |
| **Stockpiles** | Industry stockpiles (`ScriptIndustry::GetStockpiledCargo`) and station cargo (`ScriptStation::GetCargoWaiting`) | **Native** |
| **Production buildings** | NewGRF industries — **128 types per GRF** (`industry_type.h:46`), each with accepted/produced cargo, multi-tile layouts, animation | **Native** |
| **Numeric job assignment** | `ScriptIndustry::SetProductionLevel(id, level, show_news, text)` — player sets "12 foragers", GS sets the forage hut's production level | **Native — this is the whole mechanic** |
| **Professions cap / labour pool** | GS arithmetic over `ScriptTown::GetPopulation()`, distributed across industries via production levels | **Native** |
| **Caravans, hunting parties** | Road vehicles via NewGRF. Capacity, speed, running cost, cargo refit all definable. | **Native** |
| **Trails** | Roads, reskinned. Rivers/canoes → ships if you want a second movement class. | **Native** |
| **Technology gating of caravan/party types** | **`ScriptEngine::EnableForCompany()` / `DisableForCompany()`** (`script_engine.hpp:312,323`, `@api -ai` → GS-only). Disable every vehicle at game start, enable as techs unlock. | **Native — the key finding** |
| **Tech tree state, research progress** | GS Squirrel state, persisted via the script's `Save()`/`Load()` methods (`script_instance.cpp:537`) | **Native** |
| **Gating buildings by tech** | Set `construction.raw_industry_construction = 0`, then GS builds industries as deity (`ScriptIndustryType::BuildIndustry`) only for unlocked types | **Native** |
| **Placing non-industry structures** (cairns, totems, palisades) | `ScriptObjectType::BuildObject()` + NewGRF Objects feature | **Native** |
| **Terrain manipulation** | `ScriptTile::RaiseTile/LowerTile/LevelTiles/DemolishTile/PlantTree/PlantTreeRectangle` | **Native** |
| **Forest regrowth** | OpenTTD's own `TileLoop_Trees` (`tree_cmd.cpp:836`) already simulates growth, density, and spread | **Native, free** |
| **Disable rail / air / ships** | `vehicle.max_trains` / `max_aircraft` / `max_ships` = 0, settable at runtime by `ScriptGameSettings::SetValue()` | **Native** |
| **Total graphics overhaul** | NewGRF **Action 0x0A** (`SpriteReplace`, `newgrf/newgrf_acta.cpp:36`) replaces any base sprite by index — terrain at 3924–4550+, GUI, everything. Plus **Action 0x05** blocks (foundations, shores, one-way roads, OpenTTD GUI icons, overlay rocks, palette). Plus per-feature GRFs for houses, industries, vehicles, objects, cargo. | **Native** |
| **Renaming everything** ("Train"→"Caravan", "Coal"→"Flint") | NewGRF pseudo-feature **`OriginalStrings = 0x48`** (`newgrf.h:107`) replaces the engine's own strings | **Native** |
| **Events to react to** | 32 event types incl. `ET_INDUSTRY_OPEN/CLOSE`, `ET_TOWN_FOUNDED`, `ET_STATION_FIRST_VEHICLE`, `ET_VEHICLE_LOST`, `ET_ENGINE_AVAILABLE`, `ET_STORYPAGE_BUTTON_CLICK`, `ET_STORYPAGE_TILE_SELECT`, `ET_GOAL_QUESTION_ANSWER` | **Native** |
| **GS acting as the player** | `ScriptCompanyMode` — *"All actions performed within the scope of this mode will be executed on behalf of the company you switched to… like the real player is executing the commands."* GS can build vehicles, roads, stations. | **Native** |

The single most useful discovery: **`ScriptEngine::DisableForCompany`/`EnableForCompany` are
GameScript-only APIs.** They exist precisely so a scenario script can control what the player
may build, independent of introduction dates. Your Civ-style tech tree gating caravan types,
hunting parties, and boats is a direct, supported use of an existing API.

---

## 4. The architecture

Three artifacts, no C++ changes:

```
lithic-newgrf/          NML source → .grf
  cargo/                ~20 cargo types: labels, weights, payment curves, town effects
  industries/           ~40 building types: forage hut, knapping floor, tannery, drying rack…
  vehicles/             caravans, hunting parties, canoes — as road vehicles / ships
  objects/              cairns, totems, palisades, firepits
  houses/               settlement dwellings by tech tier
  gfx/                  terrain (Action A), GUI icons (Action 5 0x15), foundations, shores
  strings/              Action 0x48 — rewrite the engine's own vocabulary

lithic-gamescript/      Squirrel
  main.nut              tick loop, slice scheduling
  settlement.nut        population, labour pool, consumption, aggregate ledger
  professions.nut       job allocation → SetProductionLevel per industry
  tech.nut              tech graph, research accumulation, Enable/DisableForCompany
  ui/                   story pages: settlement panel, research panel, alerts
  world.nut             resource seeding, forage regrowth nudges, wildlife
  save.nut              Save()/Load() state serialisation

lithic-scenario/        starting world
  heightmap + scenario, settings preset (max_trains=0, raw_industry_construction=0, …)
```

**The control loop:**

```
Player                          GameScript (deity)                Engine
──────                          ──────────────────                ──────
builds trails, caravans,   ───▶                            ───▶  YAPF paths caravans
sets caravan orders                                              cargo moves, stations fill

clicks [+ Foragers] on     ───▶ ET_STORYPAGE_BUTTON_CLICK
the settlement story page       → reallocate labour
                                → SetProductionLevel(forage_hut, n)
                                → UpdateElement(text)      ───▶  industry output changes

                           ◀─── SetText(town, ledger)      ◀───  reads population,
                                                                 stockpiles, production

delivers food to the       ───▶                            ───▶  TAE_FOOD grows the town
settlement                                                       natively

                                research accrues from work
                                → EnableForCompany(travois) ───▶  new caravan buildable
                           ◀─── ScriptNews: "You have learned…"
```

**Division of labour:** the engine does transport, pathfinding, cargo physics, rendering, and
town growth. The GameScript does the settlement economy, professions, and technology. NewGRF
does identity — art, content definitions, and vocabulary. Nothing needs C++.

---

## 5. The hard limits

Four real constraints. The first is the one that matters.

### 5.1 No custom GUI — this is the binding constraint

GameScript's *entire* output surface:

| Surface | Capability |
|---|---|
| **Story pages** (`ScriptStoryPage`) | Text blocks, location links, goal refs, and **three button kinds**: `SPET_BUTTON_PUSH` (fires `ET_STORYPAGE_BUTTON_CLICK`), `SPET_BUTTON_TILE` (player picks a tile), `SPET_BUTTON_VEHICLE` (player picks a vehicle). Buttons have colour, left/right float, and a cursor. `UpdateElement()` rewrites text live. |
| **Goals list** (`ScriptGoal`) | Persistent objectives with progress |
| **Modal questions** (`ScriptGoal::Question`) | A dialog with text and up to **18 predefined buttons** (Yes/No/OK/Cancel/Accept/Decline/Retry/Previous/Next/Stop/Start/Go/Continue/Restart/Postpone/Surrender/Ignore/Close) |
| **News items** (`ScriptNews`) | Notifications |
| **Town / industry text** (`SetText`) | Arbitrary text injected into the existing town and industry windows |
| **League table** (`ScriptLeagueTable`) | A sortable scoreboard — repurposable as a stats table |
| **Signs, viewport** | Map annotations, scroll-to |
| **Widget highlight** (`ScriptWindow::Highlight`) | Highlight a widget in *any* existing window; clicking it fires `ET_WINDOW_WIDGET_CLICK` — but the highlight clears on click (`window.cpp:755`), so it's a tutorial nudge, not an input surface |

**What you do not get:** custom windows, sliders, numeric entry fields, tables, dropdowns,
tooltips, drag-and-drop, or any layout control beyond paragraph flow with floated buttons.

So your settlement management screen is a story page like:

```
   THE SETTLEMENT AT ELDER SPRING          Population 47  ·  Idle 6

   Foragers          12    [ − ] [ + ]
   Woodcutters        8    [ − ] [ + ]
   Knappers           5    [ − ] [ + ]
   Hunters            6    [ − ] [ + ]
   Tanners            4    [ − ] [ + ]
   Potters            3    [ − ] [ + ]
   Builders           3    [ − ] [ + ]

   Stores:  Berries 340 ▲   Meat 88 ▼   Firewood 12 ⚠   Flint 210 ▬
```

Every `[−]`/`[+]` is a push button; the numbers are text refreshed by `UpdateElement()`.
**This genuinely works** — and it is genuinely clunky. Two clicks to move one worker, no
click-and-hold repeat, a page rebuild per change, and no way to show a graph or a sortable
table. For a game whose core interaction *is* job assignment, that friction lands directly on
your core loop. Prototype this early ([§7 Phase 1](#7-build-plan)) and judge it with your own
hands before committing.

### 5.2 Squirrel opcode budget

```
script.script_max_opcode_till_suspend   default 10000   min 500   max 250000   (NewgameOnly)
script.script_max_memory_megabytes      default 1024    max 8192
```

10,000 interpreted opcodes per tick by default. Squirrel is slow; that's a few hundred
meaningful operations plus loop overhead. You **cannot** iterate 300 vehicles or 40 industries
every tick.

The engine's model helps: scripts run cooperatively and suspend, with `ScriptController::Sleep(ticks)`,
`GetOpsTillSuspend()` for self-throttling, and `DecreaseOps()` to charge yourself. Design for
**explicit slice scheduling** — the same discipline as
[guidelines §3](03-game-logic-guidelines.md#3-amortise-everything-never-iterate-everything):
recompute the ledger once per day, reallocate labour on change only, touch a few industries per
tick. `script_max_opcode_till_suspend` is `NewgameOnly`, so raise it in your shipped settings
preset rather than expecting to change it at runtime.

### 5.3 Money cannot be removed

`Money` and `CommandCost` are C++ core. From the mod layer you can:

- Set all NewGRF construction and vehicle costs to near-zero, and cargo `initial_payment` low
  and flat (`transit_periods` tuned so short hauls aren't punished).
- Have GS top up the balance with `ScriptCompany::ChangeBankBalance()` and cap the loan with
  `SetMaxLoanAmountForCompany()`.
- Rename "money" via Action 0x48 to something diegetic — *stored provisions*, *obligation*,
  *standing* — and treat it as one genuine abstract resource rather than hiding it.

You cannot delete the finances window, the company value graph, or the loan machinery. The
honest options are *repurpose* (best) or *neutralise and ignore* (fine). Fighting it is wasted
effort.

Note the related economic mismatch: cargo payment scales with **distance and transit time**, so
short supply lines pay poorly — backwards for a village game. Cargo `initial_payment` and
`transit_periods` are per-cargo NewGRF properties, so you can flatten the curves, but the
gradient never fully disappears.

### 5.4 GameScript cannot veto player actions

There is no hook to reject a player command. Gating works by **not offering** the thing:

- Vehicles → `DisableForCompany` (clean, native).
- Industries/buildings → set `construction.raw_industry_construction = 0` and have GS build on
  request via a `SPET_BUTTON_TILE` ("choose where to raise the tannery"). This is *better* UX
  than the fund-industry window anyway.
- Terrain and track building → cannot be restricted. The player can always terraform and lay
  trails wherever. Accept it; it's TTD-shaped, which is what you want.

---

## 6. Workable contortions

| Want | Approach | Verdict |
|---|---|---|
| **Seasons with teeth** | `LandscapeType::Arctic` gives a snow line; GS can vary industry production levels and settlement consumption on a yearly cycle, and post news | Good enough; no per-tile weather |
| **Hunting parties that deplete game** | Hunting hut = industry producing meat/hide; GS reduces its production level as nearby forest is cleared (count tree tiles via `ScriptTile`), restores it as trees regrow | Convincing, cheap |
| **Foraging that exhausts** | Same pattern — production level as a function of surrounding untouched terrain | Convincing |
| **Chopping actually clears forest** | `ScriptTile::DemolishTile` on tree tiles near an active woodcutter's lodge; regrowth is engine-native | Native and satisfying |
| **Research from work** | GS accumulates knowledge from monthly transported/produced totals (`GetLastMonthProduction`, `GetLastMonthTransported`), then unlocks | Straightforward |
| **Population caps by housing** | `SetGrowthRate` / `ExpandTown` throttled against GS's own housing count | Fine |
| **Famine and decline** | GS lowers growth rate, posts news, and can `SetProductionLevel` down; town population shrinkage is engine-driven when unfed | Partial — the engine's decline model is coarse |
| **Multiple settlements** | One `Town` each, one story page each | Native |
| **A stats table** | `ScriptLeagueTable` repurposed | Ugly but real |

---

## 7. Build plan

Estimates assume one developer plus art. Contrast with the fork's multi-year plan.

| # | Phase | Duration | Deliverable / gate |
|---|---|---|---|
| 0 | **Toolchain** — NML + `grfcodec`/`nforenum`, GS skeleton, a hot-reload loop (rebuild GRF, restart scenario) | 1 wk | Placeholder GRF loads; GS logs a tick |
| 1 | **UI spike — do this first** | **1–2 wk** | A story page with `[−]/[+]` per profession, live-updating text, driving `SetProductionLevel` on 3 dummy industries. **Judge the clunkiness with your own hands. This decides §8.** |
| 2 | **Cargo & content skeleton** — ~20 cargo types, ~10 industries, 2 caravan types, payment curves flattened | 2–3 wk | Player can haul flint from a knapping floor to a settlement |
| 3 | **Settlement core** — population, labour pool, consumption, aggregate ledger, `SetText` display, `TAE_FOOD` growth | 3–4 wk | Settlement grows when fed, shrinks when not |
| 4 | **Professions** — allocation, skill/efficiency, idle handling, `SetProductionLevel` wiring | 2–3 wk | Job assignment fully drives the economy |
| 5 | **Technology** — tech graph as data, research accumulation, `Enable/DisableForCompany`, unlock-gated GS industry construction, research story page | 3–4 wk | Tech tree gates caravans, buildings, and recipes |
| 6 | **Graphics overhaul** — terrain via Action A, GUI icons via Action 5 0x15, all industries/vehicles/houses/objects, full Action 0x48 vocabulary rewrite | 8–12 wk (art-bound) | It looks and reads like a Stone Age game, not OpenTTD |
| 7 | **World & scenario** — heightmap or generated maps, resource seeding, settings preset, starting state | 2–3 wk | A new game starts correctly with no manual setup |
| 8 | **Depletion & seasons** — forage/game exhaustion, forest clearing, yearly cycle | 2–3 wk | The world responds to exploitation |
| 9 | **Balance & polish** — opcode profiling, GS slice tuning, news/alerts, tutorial via `ScriptWindow::Highlight` | 4–6 wk | Playable for hours without confusion |

**Roughly 7–10 months to a playable, distinctive game** — against multiple years for the fork.
Phase 6 is the long pole and it's art, not engineering, which is a much healthier risk profile.

Carry over from the guidelines, all still applicable: keep the GS deterministic and
integer-only, amortise with explicit slices, keep every balance number in a data table,
version your `Save()`/`Load()` state, and soak-test headlessly.

---

## 8. Recommendation: mod first, then a thin patch

**Take the mod route.** It's dramatically cheaper, it keeps your content proprietary
([§2](#2-licensing--the-sleeper-advantage)), you inherit upstream OpenTTD improvements forever,
and the design maps onto native mechanics to a degree that is genuinely surprising — settlement
growth from food delivery and tech gating of vehicle types are both *configuration*, not code.

**But run the Phase 1 UI spike before anything else**, because §5.1 is the one constraint that
lands on your core loop. Two outcomes:

- **The story-page UI is tolerable** → stay pure-mod. Ship a NewGRF + GameScript + scenario,
  optionally bundled with unmodified OpenTTD. Nothing further needed.
- **It isn't** (my expectation, honestly, for a game whose central interaction is numeric job
  assignment) → **go hybrid, not full fork.** Maintain a small patch set against upstream
  OpenTTD:

  | Patch | Size | Buys you |
  |---|---|---|
  | A GS-scriptable settlement window — a real widget window with rows, `[−]/[+]` steppers, and numeric entry, driven by a GS-supplied table | ~1.5–3k LOC | Fixes the core-loop friction |
  | Extend `ScriptStoryPage` with a numeric-input element and a value-slider | ~500 LOC | Cheaper alternative to the above |
  | Hide/disable the money UI and zero the economy properly | ~300 LOC | Removes the deepest thematic awkwardness |
  | Raise `script_max_opcode_till_suspend` above 250k, or add a GS profiling hook | ~100 LOC | Headroom if Phase 3–4 profiling demands it |

  That's **~2–4k lines of C++ against a clean upstream**, versus the fork's ~167k deleted and
  ~61k rewritten. All of it is additive — new windows and new API surface, not surgery on
  `viewport.cpp` — so rebasing on upstream stays cheap. And the first two are plausibly
  upstreamable, which would eliminate the patch set entirely.

  The licensing cost of going hybrid is real and should be priced in: **the moment you ship a
  modified binary, that binary is GPL v2 and its source must be published.** Your NewGRF and
  GameScript remain separate works, so only the engine patches become public — a far smaller
  concession than the full fork, but not zero.

**On the earlier assessments.** Docs 01–04 remain accurate for the Banished-style design they
assessed; this pivot is a different game, and a much cheaper one. The two documents worth
re-reading against it are
[guidelines §3 (amortisation)](03-game-logic-guidelines.md#3-amortise-everything-never-iterate-everything)
and [§8 (content is data)](03-game-logic-guidelines.md#8-content-is-data-not-code) — both apply
verbatim to Squirrel, and the opcode budget makes them *more* binding, not less.

**What you give up by pivoting** — worth stating plainly so the choice is informed: the
texture of watching individual people work. Banished's appeal is substantially in seeing a
named villager walk to a tree, chop it, and carry logs home. Aggregate numbers plus caravans is
a different pleasure — closer to *Transport Tycoon with a settlement to supply*, or to
Anno/Settlers at arm's length. That is a coherent and appealing game, and it is *deliverable*
where the other was a multi-year bet. Just don't expect the pivot to be free of that cost.

---

*Assessed against OpenTTD `be4f099` (2026-08-10). All API claims verified against
`src/script/api/*.hpp`, NewGRF capabilities against `src/newgrf/newgrf_act5.cpp` and
`newgrf_acta.cpp`, and limits against `src/table/settings/script_settings.ini`,
`src/cargo_type.h`, and `src/industry_type.h`.*
