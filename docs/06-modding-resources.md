# OpenTTD modding resources

Curated for the [aggregate-design route](05-aggregate-design-modding-route.md) — a NewGRF total
conversion plus a GameScript. Ordered by what this project actually needs, not alphabetically.
Every link below was checked live in August 2026.

**One structural warning before you start.** The official NML tutorial teaches vehicles and
objects and **does not cover industries, cargoes, or houses** — which will be the bulk of your
content. The working path is: *tutorial for syntax → reference pages for industries/cargoes →
read a real industry set's source.* Budget for that third step; it's where the actual learning
happens.

---

## Contents

- [1. Start here](#1-start-here)
- [2. The two most valuable resources](#2-the-two-most-valuable-resources)
- [3. NewGRF — content and graphics](#3-newgrf--content-and-graphics)
- [4. GameScript — the game logic](#4-gamescript--the-game-logic)
- [5. Reference codebases to read](#5-reference-codebases-to-read)
- [6. Graphics tooling](#6-graphics-tooling)
- [7. Community](#7-community)
- [8. A two-week ramp-up](#8-a-two-week-ramp-up)

---

## 1. Start here

| Resource | Why |
|---|---|
| [NML Tutorial (TT-Wiki)](https://www.tt-wiki.net/wiki/NMLTutorial) | The canonical entry point. 21 chapters as one continuous story with prev/next links. Covers install, graphics files, language files, syntax, spritesets/spritegroups, templates, cargotable, callbacks and switch blocks, road vehicles, trams, trains, **objects**, **32bpp sprites**, and **base graphics replacement** (ch. 20 — directly relevant to your reskin). |
| [NML reference — `NML:Main`](https://newgrf-specs.tt-wiki.net/wiki/NML:Main) | The complete language reference. This is where you live after the tutorial. |
| [OpenTTD Development Tools](https://wiki.openttd.org/en/Development/NewGRF/Development%20Tools) | The official index of every NewGRF toolchain option, kept current. |
| [Game Script manual](https://wiki.openttd.org/en/Manual/Game%20script) | What GameScripts can do, where to install them (`$OpenTTD/game/yourGSname` for development, **not** the online-content directory), and links to a minimal starter template. |
| [Script Development — Introduction](https://wiki.openttd.org/en/Development/Script/Introduction) | The GS/AI development hub: [Basics](https://wiki.openttd.org/en/Development/Script/Basics), [`info.nut`](https://wiki.openttd.org/en/Development/Script/AIInfo), [`main.nut`](https://wiki.openttd.org/en/Development/Script/AIMain), [Save and Load](https://wiki.openttd.org/en/Development/Script/Save%20and%20Load), [Lists](https://wiki.openttd.org/en/Development/Script/Lists), [Squirrel pitfalls](https://wiki.openttd.org/en/Development/Script/Squirrel), [Things you need to know](https://wiki.openttd.org/en/Development/Script/Need%20To%20Know). Written for AIs, but the wiki's own advice is that it applies to GameScripts too. |
| [GameScript API reference](https://docs.openttd.org/gs-api/) | Auto-generated from source, every `GS*` class. The build I assessed against (`be4f099`) is the one currently published there, so the docs and [doc 05](05-aggregate-design-modding-route.md) agree exactly. |

Read [Save and Load](https://wiki.openttd.org/en/Development/Script/Save%20and%20Load) early, not
late — it's the mechanism behind [Seam 2](05-aggregate-design-modding-route.md#seam-2--gamescript-state-stays-authoritative)
(GameScript-authoritative state), and retrofitting it is painful.

---

## 2. The two most valuable resources

**Your core mechanic already exists, twice, in pure GameScript.** "Deliver cargo to a settlement,
it grows; under-supply it, it shrinks" is exactly what city-builder GameScripts do. Read both of
these before writing a line of your own.

### [CityBuilder](https://github.com/AphidGit/CityBuilder) — the closer match

16 `.nut` files. Implements custom town growth formulas with multipliers and thresholds,
**per-population cargo requirements**, cargo introduction delays as industries develop,
**warehouse storage with configurable decay rates**, town absorption and regrowth, goals and
story progression, 40+ configurable settings, and four game modes.

The warehouse-with-decay and per-population cargo demand systems are close to what
[doc 05 §3](05-aggregate-design-modding-route.md#3-what-maps-natively) proposes for settlement
stores and consumption. This is the single most useful codebase for you.

### [SimpleCB](https://github.com/TheDude-gh/simplecb) — the cleaner one to learn from

`main.nut`, `classes.nut`, `town.nut`, `saveload.nut` — a much smaller, more legible
implementation of the same idea. Towns grow when supplied and shrink when not; cargo becomes
mandatory at population thresholds; two growth mechanisms (OpenTTD's native growth vs. the script
placing houses itself); progress via the storybook. Also ships economy presets for FIRS, ECS,
YETI and XIS — a worked example of writing one script against several cargo sets, which is the
same problem as writing one script against *your* evolving cargo set.

Read `saveload.nut` specifically. GPL-2.0, 95 commits.

Two more worth skimming:

- [Villages Is Villages](https://github.com/mattkimber/openttd_villages_is_villages) — town
  behaviour control, by a well-regarded NewGRF author.
- [OpenTTDStorybookGS](https://github.com/SarahRoseLives/OpenTTDStorybookGS) and
  [IntroGameTool](https://github.com/nielsmh/IntroGameTool) — small, focused story-page examples.
  Useful for [Phase 1's UI spike](05-aggregate-design-modding-route.md#95-recommended-sequencing),
  since story pages are your only input surface.

---

## 3. NewGRF — content and graphics

### The pages you'll need most

The tutorial skips all of these, and they're where your content lives:

| Page | For |
|---|---|
| [`NML:Industries`](https://newgrf-specs.tt-wiki.net/wiki/NML:Industries) | **Your buildings.** Complete reference: tile layouts, lifecycle types (extractive / organic / processing / black hole), funding costs, and the modern `accept_cargo()` / `produce_cargo()` array form (NML 0.5+). Production levels run **4–128**. Callbacks incl. `produce_cargo_arrival`, `produce_256_ticks`, `location_check`, `monthly_prod_change`, `build_prod_change`. **Limit: at most 16 accepted and 16 produced cargo labels per industry.** |
| [`NML:IndustryTiles`](https://newgrf-specs.tt-wiki.net/wiki/NML:IndustryTiles) | Per-tile graphics, animation, acceptance |
| [`NML:Cargos`](https://newgrf-specs.tt-wiki.net/wiki/NML:Cargos) | **Your resources.** Weight, payment curves, transit periods, capacity multiplier, and the town-effect property that makes food delivery grow a settlement |
| [`NML:Cargotable`](https://newgrf-specs.tt-wiki.net/wiki/NML:Cargotable) | Declaring the cargo label set |
| [`NML:Houses`](https://newgrf-specs.tt-wiki.net/wiki/NML:Houses) | Settlement dwellings by tech tier |
| [`NML:Objects`](https://newgrf-specs.tt-wiki.net/wiki/NML:Objects) | Cairns, totems, palisades, firepits — anything placed but not an industry |
| [`NML:Base_Graphics`](https://newgrf-specs.tt-wiki.net/wiki/NML:Base_Graphics) | **Terrain and base sprite replacement** — the heart of your visual overhaul |
| [`NML:Spriteset`](https://newgrf-specs.tt-wiki.net/wiki/NML:Spriteset) / [`Spritelayout`](https://newgrf-specs.tt-wiki.net/wiki/NML:Spritelayout) / [`Spritegroup`](https://newgrf-specs.tt-wiki.net/wiki/NML:Spritegroup) | How sprites attach to anything |
| [`NML:Getting_started`](https://newgrf-specs.tt-wiki.net/wiki/NML:Getting_started) / [`Block_syntax`](https://newgrf-specs.tt-wiki.net/wiki/NML:Block_syntax) | Language mechanics |
| [Action 5 spec](https://newgrf-specs.tt-wiki.net/wiki/Action5) | The replaceable sprite blocks (foundations, shores, GUI icons, overlay rocks, palette) — the low-level view of what [doc 05 §3](05-aggregate-design-modding-route.md#3-what-maps-natively) lists |

### The compiler

- [NML on GitHub](https://github.com/OpenTTD/nml) — source and issues
- [`nml` on PyPI](https://pypi.org/project/nml/) — `pip install nml`, the normal way to get it
- [NFO spec (GRFSpecs)](https://newgrf-specs.tt-wiki.net/wiki/Main_Page) — the byte-level format
  NML compiles to. You won't write NFO, but you'll read it in error messages.
- [GRFCodec / NFORenum](https://www.openttd.org/downloads/grfcodec-releases/latest) — encoder and
  linter; needed for some workflows and for decoding other people's GRFs
- [YAGL](https://github.com/UnicycleBloke/yagl/tree/master/docs) — decompiles existing GRFs into
  readable text. Genuinely useful for answering "how did *they* do that?"
- [Grf2Html](http://dev.openttdcoop.org/projects/grf2html) — renders a GRF as browsable HTML for
  study

---

## 4. GameScript — the game logic

Beyond the wiki pages in [§1](#1-start-here):

- **[OpenTTD GameScript Guide](https://www.openttdgsguide.uk/)** — a dedicated third-party GS
  tutorial site. Its landing page is JavaScript-rendered so I couldn't read the table of contents
  through a fetch; open it in a browser and judge it yourself. Worth ten minutes given how thin
  the official GS-specific material is.
- **[GameScripts on BaNaNaS](https://bananas.openttd.org/package/game-script)** — every published
  GameScript. Most are small and readable; browsing for one that solves a problem you have is
  usually faster than reasoning from the API docs.
- **[Squirrel language docs](https://wiki.openttd.org/en/Development/Script/Squirrel)** — the
  OpenTTD-specific pitfalls page matters more than the upstream Squirrel manual. Read it before
  debugging something weird.
- **[Script libraries](https://wiki.openttd.org/en/Development/Script/Library)** — importable
  modules; check here before writing utility code.

Two things to internalise early, both from [doc 05](05-aggregate-design-modding-route.md):

1. **Declare an API version in `info.nut`.** OpenTTD ships 14 GameScript compatibility layers
   (`bin/game/compat_*.nut`) precisely so old scripts keep working. Pin a version deliberately.
2. **The opcode budget is real** — 10,000 per tick by default. `GSController.GetOpsTillSuspend()`
   and `GSController.Sleep()` are your throttling tools, and
   [guidelines §3](03-game-logic-guidelines.md#3-amortise-everything-never-iterate-everything)
   applies verbatim to Squirrel.

---

## 5. Reference codebases to read

Reading a shipped, mature set teaches more than any tutorial. In order of usefulness to you:

| Project | What it teaches |
|---|---|
| **[FIRS](https://github.com/andythenorth/firs)** (GPL-2.0, 7,416 commits, active) | **The canonical industry set.** Dozens of industries and cargoes, full production chains, economy variants. If you want to know how to structure ~40 industries and ~20 cargoes without going mad, this is the answer. **Caveat:** FIRS generates NML from Python templates rather than writing NML directly. That's the right call at its scale and it may be right at yours, but it means you can't read it as plain NML — learn NML first, then read FIRS. |
| [AXIS](https://github.com/EmperorJake/AXIS) and [AIRS](https://github.com/andybiotic/airs_andysine) | Forks of FIRS. Useful as diffs — seeing what someone changed and why is often clearer than the original. |
| [BRIX](https://blog.openttdcoop.org/2017/10/23/brix-0-0-2-is-here/) | A **purely visual** graphics-replacement NewGRF adding 32bpp and 4× zoom while keeping 8bpp compatibility. The closest existing model for your visual overhaul, and proof the approach works at scale. |
| OpenGFX | The full base-set replacement — **~6,990 sprites**. Cite that number when scoping your art budget; it is the realistic size of "replace everything," and it's why [doc 05](05-aggregate-design-modding-route.md#7-build-plan) puts the graphics phase at 8–12 weeks and calls it art-bound. |
| [Graphics Replacement (wiki)](https://wiki.openttd.org/en/Archive/Community/Graphics%20Replacement) and [Playing with 32bpp graphics](https://wiki.openttd.org/en/Community/NewGRF/Playing%20with%2032%20bpp%20graphics) | Background on how base-graphics replacement actually works in practice |
| [NewGRF FAQ](https://wiki.openttd.org/en/Archive/Community/NewGRF%20FAQ) | Archived but still the fastest answer to many beginner questions |

---

## 6. Graphics tooling

Isometric pixel art at OpenTTD's exact projection is its own craft. The community solved the
tooling problem with voxels:

- **[PixelTool](https://www.tt-forums.net/viewtopic.php?f=26&t=69974)** — voxel-based sprite
  editor built for this projection
- **[Voxel workflow video series](https://www.youtube.com/playlist?list=PLuO1EdT1BRkdgicGIvQkV4QaBDqtsYmBs)**
  — MagicaVoxel → OpenTTD sprites. **Watch this before hiring or briefing an artist.** Modelling
  once in voxels and rendering all 8 rotations and 6 zoom levels is dramatically cheaper than
  hand-drawing each, and it's how most modern OpenTTD art is made.
- **[TTDViewer](https://github.com/frosch123/TTDViewer)** — previews `.pcx`/`.png` with palette
  animation and recolouring, so you can check company-colour remapping before shipping
- **[GRF Wizard](http://www.andreszsogon.com/grf-wizard/)** — GUI for grfcodec, palette conversion
- [m4nfo](http://www.ttdpatch.de/grfspecs/m4nfoManual/index.html) — macro-based NFO frontend; an
  alternative to NML if you dislike it
- [GRFMaker](http://users.tt-forums.net/grfmaker/) — GUI GRF creation, **unmaintained**; listed
  for completeness only

---

## 7. Community

The forums are where the actual expertise is, and the NewGRF authors are responsive:

- [TT-Forums — NewGRF Development](http://www.tt-forums.net/viewforum.php?f=26)
- [TT-Forums — NewGRF Technical Discussion](http://www.tt-forums.net/viewforum.php?f=68)
- [#openttdcoop Dev Zone](http://dev.openttdcoop.org) — hosts several major projects
- [BaNaNaS: NewGRFs](https://bananas.openttd.org/package/newgrf) and
  [Game Scripts](https://bananas.openttd.org/package/game-script) — the content library; browse it
  to see what has been done and to download things to decompile
- OpenTTD's Discord has dedicated `#newgrf`, `#open-graphics`, and development channels — linked
  from the [project README](https://github.com/OpenTTD/OpenTTD)

Worth telling them what you're building early. A Stone Age total conversion is unusual enough that
people will be interested, and the NewGRF veterans will save you weeks on questions you don't yet
know to ask.

---

## 8. A two-week ramp-up

Concretely, before Phase 0 of [the build plan](05-aggregate-design-modding-route.md#7-build-plan):

**Days 1–3 — NewGRF basics.** `pip install nml`. Work the
[NML tutorial](https://www.tt-wiki.net/wiki/NMLTutorial) start to finish, including ch. 16
(objects), 19 (32bpp) and 20 (base graphics replacement). Ship one ugly object into a running game.

**Days 4–5 — content definitions.** Read [`NML:Cargos`](https://newgrf-specs.tt-wiki.net/wiki/NML:Cargos),
[`NML:Cargotable`](https://newgrf-specs.tt-wiki.net/wiki/NML:Cargotable) and
[`NML:Industries`](https://newgrf-specs.tt-wiki.net/wiki/NML:Industries). Define two cargoes and
one industry that turns one into the other. This is your whole economy in miniature.

**Days 6–8 — GameScript.** Read [Basics](https://wiki.openttd.org/en/Development/Script/Basics),
[`info.nut`](https://wiki.openttd.org/en/Development/Script/AIInfo),
[`main.nut`](https://wiki.openttd.org/en/Development/Script/AIMain),
[Save and Load](https://wiki.openttd.org/en/Development/Script/Save%20and%20Load) and the
[Squirrel pitfalls](https://wiki.openttd.org/en/Development/Script/Squirrel) page. Then read
[SimpleCB](https://github.com/TheDude-gh/simplecb) end to end — it's small enough to hold in your
head. Write a GS that reads a town's population and shows it on a story page.

**Days 9–10 — the real prototype.** Wire the story page to `GSIndustry.SetProductionLevel()` on
your day-4 industry via push buttons. **This is [Phase 1's UI spike](05-aggregate-design-modding-route.md#7-build-plan),
and it's the decision point for the whole project** — pure mod versus hybrid.

**Days 11–14 — study.** Read [CityBuilder](https://github.com/AphidGit/CityBuilder)'s cargo-demand
and warehouse code. Skim [FIRS](https://github.com/andythenorth/firs) for structure, not detail.
Watch the [voxel workflow series](https://www.youtube.com/playlist?list=PLuO1EdT1BRkdgicGIvQkV4QaBDqtsYmBs)
and decide your art pipeline before you commission anything.

You end the fortnight with the syntax, a working toolchain, an answer to the one question that
decides the architecture, and an informed art plan.

---

*Links verified August 2026. GameScript API reference published from OpenTTD `be4f099` — the same
commit the assessments in this repo were made against.*
