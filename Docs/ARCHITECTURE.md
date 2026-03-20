# Game Engine Architecture Reference

A living reference document covering all architectural decisions made during the design of this data-driven MMO game engine. Intended as a guide for contributors working on both the C# engine and YAML content.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Core Philosophy](#2-core-philosophy)
3. [File Structure](#3-file-structure)
4. [The Engine Layer](#4-the-engine-layer)
5. [The Effect System](#5-the-effect-system)
6. [The Rule System](#7-the-rule-system)
7. [The System Loader](#7-the-system-loader)
8. [Content Registry](#8-content-registry)
9. [Game State](#9-game-state)
10. [The Map System](#10-the-map-system)
11. [The Render Snapshot](#11-the-render-snapshot)
12. [YAML Schemas](#12-yaml-schemas)
13. [Contributor Patterns](#13-contributor-patterns)
14. [What Stays in C# vs YAML](#14-what-stays-in-c-vs-yaml)

---

## 1. Project Overview

An auto-battler MMO with OSRS-inspired skills, base building, roguelike dungeon runs, and a meta/self-aware tone. The game is designed to be playable in a browser (web-first) with an optional desktop client. Both clients share the same engine and content — only the rendering layer differs.

### Core game systems

- **Skills** — crafting, magic, and combat skills with OSRS-style leveling and milestones
- **Base building** — player base with NPC workers, buildings, and passive resource generation
- **Combat** — semi-active: auto-resolves but player can use abilities and consumables
- **Roguelike dungeons** — procedurally generated runs with rewards feeding back into the base
- **Static world map** — tile-based, authored in YAML, areas linked via transition tiles
- **Meta layer** — snarky tooltips, game-aware NPC dialogue, fourth-wall humor

---

## 2. Core Philosophy

> The engine executes primitives. It never knows what they mean.

Every architectural decision flows from this principle. Game concepts — skills, recipes, doors, quests, NPCs — live entirely in YAML and in registered rule functions. The engine core has no knowledge of any of them.

### The three layers

```
┌─────────────────────────────────────────────────────┐
│                   YAML CONTENT                       │
│  items · skills · maps · tilesets · systems · npcs  │
│  All game rules, definitions, and behaviors here    │
└────────────────────┬────────────────────────────────┘
                     │ loaded by
┌────────────────────▼────────────────────────────────┐
│                  C# ENGINE CORE                      │
│  GameState · EventBus · EffectExecutor               │
│  RuleRegistry · SystemLoader · ContentRegistry       │
│  Knows: set, add, emit, call, dot-paths, events     │
│  Does NOT know: skills, crafting, doors, quests     │
└────────────────────┬────────────────────────────────┘
                     │ exposes state via
┌────────────────────▼────────────────────────────────┐
│                   CLIENTS                            │
│  Text renderer · Sprite renderer · Headless tests   │
│  Read RenderSnapshot · Resolve own art references   │
└─────────────────────────────────────────────────────┘
```

### Key decisions

- **Data-driven by default.** If something can be expressed in YAML, it is.
- **Dynamic definitions.** Content definitions are `Dictionary<string, object>`, not typed C# classes. Adding a new field to any YAML file requires zero C# changes.
- **Hybrid rule system.** Game systems are defined in YAML (trigger, rules, effects). Rule *logic* lives in small registered C# functions. Adding a new game system = new YAML file. Adding a new rule type = one C# function + one `Register()` call.
- **Event-driven architecture.** Systems never call each other. Everything communicates through the event bus. A system emits an event; other systems subscribe to it.
- **Client-side rendering.** The engine produces a `RenderSnapshot` carrying raw IDs and positions. Clients resolve art references from their own asset cache. The engine never knows how it is displayed.

---

## 3. File Structure

```
/content/
  items.yaml          Item definitions (weapons, armor, consumables, materials)
  skills.yaml         Skill definitions with XP curve and milestones
  tilesets.yaml       Tile character definitions with ASCII art and color
  maps.yaml           Static map areas with ASCII grids and legends
  entities.yaml       16x16 ASCII art for player, NPCs, and ground items
  doors.yaml          Door definitions with lock/permission requirements
  npcs.yaml           (future) NPC definitions
  quests.yaml         (future) Quest definitions
  buildings.yaml      (future) Base building definitions

/systems/
  crafting.yaml       Crafting attempt handler
  skills.yaml         XP grant handler
  doors.yaml          Door interact handler
  movement.yaml       Player movement and interact handlers
  (drop new .yaml files here to add new game systems)

/src/Engine/
  EngineCore.cs       GameState, EventBus, EffectExecutor, RuleContext,
                      RuleRegistry, SystemLoader, ContentRegistry,
                      SchemaValidator, Definition
  MapSystem.cs        TileInstance, TileGrid, MapLoader, MapRules
  RenderSnapshot.cs   LayerData, CellSnapshot, FullSnapshot, DeltaSnapshot,
                      IRenderer, SnapshotBuilder

/src/Game/
  GameSystems.cs      SharedRules, CraftingRules, DoorRules, SkillRules,
                      GameBootstrap, EngineContext
```

---

## 4. The Engine Layer

The engine is intentionally minimal. It contains exactly five concepts:

| Component | Responsibility |
|---|---|
| `GameState` | Dot-path read/write over a structured dictionary |
| `EventBus` | Publish/subscribe event routing between systems |
| `EffectExecutor` | Executes the four primitives with `$$` variable resolution |
| `RuleRegistry` | Maps rule name strings to rule functions |
| `SystemLoader` | Reads system YAML files and wires them to the event bus |

The engine imports nothing from `Game.*`. It contains no strings like `"recipe"`, `"skill"`, `"door"`, or any other game concept.

---

## 5. The Effect System

Every behavior in the game — milestone unlocks, crafting results, door interactions, tile entry — is expressed as a list of effect primitives. There are exactly four:

```
set  { path, value }         Write a value into game state at a dot-path
add  { path, value }         Append a value to a list in game state
emit { event, payload }      Fire an event onto the event bus
call { fn, args }            Invoke a named built-in function (escape hatch)
```

### How effects flow

```
YAML effect list
      │
      ▼
EffectExecutor.Execute(effects, context)
      │
      ├── Resolve $$ variables from context
      │     $$key          → payload["key"]
      │     $$key:dot.path → content.Get(inferredType, payload["key"])
      │                       .GetPath("dot.path")
      │
      ├── set  → GameState.Set(path, value)
      ├── add  → GameState.Add(path, value)
      ├── emit → EventBus.Emit(event, payload)
      └── call → _builtins[fn](args)
```

### `$$` variable resolution

Effect values starting with `$$` are resolved from the current event context before execution. The engine infers content type from the payload key name:

```
$$item_id              → payload["item_id"]
$$item_id:craft_requirement.skill
                       → content.Get("items", payload["item_id"])
                                .GetPath("craft_requirement.skill")
```

Key suffix `_id` is stripped and pluralised to infer content type: `item_id → items`, `door_id → doors`, `npc_id → npcs`.

### Example — skill milestone effects in YAML

```yaml
effects:
  - add:  { path: player.known_recipes, value: iron_sword }
  - add:  { path: player.known_recipes, value: iron_helmet }
  - emit: { event: base.worker_unlocked, payload: { worker: apprentice_smith, building: smithy } }
  - emit: { event: player.title_earned,  payload: { title_id: master_smith } }
```

The engine executes this list. It does not know what a recipe, worker, or title is.

---

## 6. The Rule System

Rules are named boolean functions registered in `GameBootstrap`. YAML system files reference them by name. When an event fires, the system's rule list is evaluated top-to-bottom. The first failing rule stops execution and fires that rule's `on_fail` effects.

### Rule function contract

```csharp
// Input:  RuleContext  (event payload + rule params + state + content)
// Output: bool         (true = pass, false = fail)
Func<RuleContext, bool>
```

### RuleContext API

```csharp
ctx.EventValue("item_id")              // read from event payload
ctx.EventValue<int>("target_x")        // typed read from payload
ctx.RuleParam<string>("source")        // read from YAML rule block params
ctx.StateValue<int>("player.skills.melee.level")  // read from game state
ctx.StateListContains("player.known_recipes", id) // check list membership
ctx.GetDefinition("items", "iron_sword")          // look up content definition
```

### Rule evaluation flow

```
System YAML rule list
        │
        ▼
RuleRegistry.EvaluateAll(ruleDefs, payload, state, content)
        │
        ├── For each rule def:
        │     Extract "check" name
        │     Extract rule params (everything except "check" and "on_fail")
        │     Construct RuleContext
        │     Call registered function → bool
        │
        ├── PASS → continue to next rule
        │
        └── FAIL → return on_fail effects to SystemLoader
                   (SystemLoader executes them + emits failure_event)
```

### Example — rule definition in YAML

```yaml
rules:
  - check: player_meets_requirement
    source: door                    # rule param — passed to ctx.RuleParam()
    on_fail:
      - emit: { event: door.blocked, payload: { reason: requirement_not_met } }

  - check: player_has_permission
    source: door
    on_fail:
      - emit: { event: door.blocked, payload: { reason: no_permission } }
```

### Shared rules

These rules are generic enough to be reused across any system via a `source:` param:

| Rule name | What it checks |
|---|---|
| `entity_exists` | A definition with the event's `{source}_id` exists in the registry |
| `player_meets_requirement` | Player meets the `requirement` block on the source definition (skill/item/stat) |
| `player_has_permission` | Player holds any permission from the source definition's `permissions` list |
| `player_has_flag` | `player.flags.{flag}` is true in game state |
| `state_contains` | A list at `path` in game state contains `value` |

---

## 7. The System Loader

A game system is a YAML file with four fields:

```yaml
on: event.name              # which event triggers this system
failure_event: event.failed # emitted if any rule fails (optional)
rules: [ ... ]              # list of rule checks to run in order
success_effects: [ ... ]    # effects to run if all rules pass
config:                     # static values available as $$key in effects
  xp_amount: 25
```

`SystemLoader.LoadAll(directory)` reads every `.yaml` file in the systems directory and subscribes each to the event bus. **Adding a new game system requires only dropping a new YAML file into `/systems/`.**

### System execution flow

```
Event fires on bus  (e.g. crafting.attempt)
        │
        ▼
SystemLoader handler (subscribed at startup)
        │
        ├── Merge config values into payload as $$key
        │
        ├── RuleRegistry.EvaluateAll(rules, payload, state, content)
        │        │
        │        ├── All pass → null returned
        │        └── Any fail → on_fail effects returned
        │
        ├── FAIL path:
        │     Execute on_fail effects
        │     Emit failure_event
        │     Return
        │
        └── PASS path:
              EffectExecutor.Execute(success_effects, payload)
```

---

## 8. Content Registry

Definitions are stored as `Dictionary<string, object>` — no typed C# model classes. Every field is dynamic. Adding a new field to any YAML file requires zero C# changes.

### Loading

```csharp
// Flat list file (items, doors, maps, tilesets, entities, npcs, quests)
content.LoadList("items", "content/items.yaml");

// Wrapped file — list is nested under a key, with top-level metadata
content.LoadWrapped("skills", "content/skills.yaml", listKey: "skills");
// Also stores xp_curve, level_cap etc. accessible via content.GetMeta()
```

### Reading definitions

```csharp
Definition item = content.Get("items", "iron_sword");    // throws if missing
Definition item = content.TryGet("items", "iron_sword"); // null if missing

item.Get<string>("name")                          // top-level field
item.GetPath<string>("craft_requirement.skill")   // nested dot-path
item.GetPath<int>("craft_requirement.level")
item.GetList<Dictionary<string,object>>("effects") // list field
item.Has("training_cost")                          // field existence check
```

### Schema validation

`SchemaValidator` enforces required fields at load time. Only required fields are listed — optional fields can appear freely in YAML without registration:

```csharp
// Required fields per content type (enforced at startup):
["items"]   = { "id", "name", "type" }
["skills"]  = { "id", "name", "category" }
["doors"]   = { "id", "name" }
["maps"]    = { "id", "name", "tileset", "grid" }
```

---

## 9. Game State

All runtime state lives in `GameState`, organised into three top-level buckets:

```
player.*    Player character state (skills, inventory, flags, permissions, titles)
base.*      Base building state (buildings, workers, upgrades, resources)
world.*     World state (current map, player position, NPC positions, ground items)
```

### Dot-path access

```csharp
state.Get("player.skills.blacksmithing.level")   // read
state.Set("player.skills.blacksmithing.xp", 150) // write
state.Add("player.known_recipes", "iron_sword")  // append to list
```

Intermediate dictionaries are created automatically by `Set`. Reads return `null` if any segment of the path is missing.

### Conventions

| Path pattern | What it stores |
|---|---|
| `player.skills.{skill_id}.xp` | Accumulated XP for a skill |
| `player.skills.{skill_id}.level` | Current level for a skill |
| `player.known_recipes` | List of recipe IDs the player can craft |
| `player.known_abilities` | List of ability IDs the player has unlocked |
| `player.inventory` | List of item IDs in player inventory |
| `player.permissions` | List of permission flags the player holds |
| `player.flags.{flag}` | Boolean flag (quest progress, one-time events) |
| `player.titles` | List of earned title IDs |
| `player.stat_bonuses.{stat}` | List of float bonuses; stat system sums these |
| `base.buildings.{building}.hireable_workers` | Workers available to hire |
| `base.buildings.{building}.available_upgrades` | Upgrades ready to purchase |
| `world.current_map` | Active map ID |
| `world.player_pos.x` / `.y` | Player tile coordinates |
| `world.maps.{map_id}.npcs.{npc_id}.x` / `.y` | NPC tile coordinates |
| `world.maps.{map_id}.ground_items.{x}_{y}.item_id` | Item on the ground |

---

## 10. The Map System

Static world maps are defined entirely in YAML and loaded into queryable `TileGrid` objects at startup.

### Two-layer tileset design

| Layer | File | Purpose |
|---|---|---|
| Tileset | `tilesets.yaml` | Defines what characters *mean* — sprite, walkability, default effects |
| Map legend | `maps.yaml` | Defines what characters mean *in this specific map* — instance data like `door_id`, `target_map` |

This separation means tile behavior is never repeated. Only instance-specific data (which door, which transition target) appears in individual map legends.

### ASCII grid format

```yaml
grid: |
  ################
  #..............#
  #..T.?.T.......#
  ##.####.#######
  ,,>A,,,,>B,,,,,
```

- Each character is one tile
- Two-character keys (`>A`, `>B`) distinguish multiple instances of the same tile type
- The engine tries two-character keys first, falls back to one character
- Spaces are void (impassable, not rendered)

### Map loading flow

```
content.LoadList("maps", "content/maps.yaml")
content.LoadList("tilesets", "content/tilesets.yaml")
        │
        ▼
MapLoader.LoadAll()
        │
        ├── For each map definition:
        │     ResolveTileset(tileset_id) → char → tile data dict
        │     ResolveLegend(map def)     → key  → merged tile data dict
        │     Parse ASCII grid line by line
        │       For each position:
        │         Try 2-char legend key, then 1-char
        │         Merge: tileset defaults ← legend overrides
        │         Create TileInstance(x, y, char, mapId, mergedData)
        │     PlaceNpcs  → world.maps.{id}.npcs.{npc_id}.{x,y}
        │     PlaceItems → world.maps.{id}.ground_items.{x_y}.{item_id,qty}
        │
        └── Store TileGrid keyed by map ID
```

### Tile capabilities

Each tile can define any of:

```yaml
walkable: true/false          # movement rule uses this
on_enter:  [ effects ]        # fires when player steps onto tile
on_interact: [ effects ]      # fires when player interacts with tile
door_id: castle_gate          # instance data — referenced by $$tile.door_id in effects
target_map: blacksmith_district  # transition destination
target_x: 7
target_y: 1
```

### Map transitions

Transition tiles (`>`, `<`) carry `target_map`, `target_x`, `target_y` in their legend entry. Their `on_enter` effect emits `map.transition`. `MapLoader` subscribes to this event and calls `ActivateMap(targetMap, tx, ty)`, which updates `world.current_map` and `world.player_pos`.

```
Player steps on > tile
        │
        ▼
movement system fires resolve_tile_and_fire_entry built-in
        │
        ▼
tile.on_enter: emit map.transition { target_map, target_x, target_y }
        │
        ▼
MapLoader.OnMapTransition handler
        │
        ▼
MapLoader.ActivateMap(targetMap, tx, ty)
  state.Set("world.current_map", targetMap)
  state.Set("world.player_pos.x", tx)
  state.Set("world.player_pos.y", ty)
  bus.Emit("map.loaded", { map_id, spawn_x, spawn_y })
```

### Movement system flow

```
Input layer emits: player.move { target_x, target_y }
        │
        ▼
systems/movement.yaml handler
        │
        ├── Rule: tile_in_bounds   → grid.InBounds(x, y)
        ├── Rule: tile_is_walkable → grid.IsWalkable(x, y)
        │
        ├── FAIL → emit movement.blocked { reason }
        │
        └── PASS:
              set world.player_pos.x = $$target_x
              set world.player_pos.y = $$target_y
              call resolve_tile_and_fire_entry { x, y }
              emit player.moved { x, y }
```

---

## 11. The Render Snapshot

The engine's only rendering responsibility is producing a `RenderSnapshot`. Clients consume it and render however they like. The engine never knows how it is displayed.

### Snapshot types

**`FullSnapshot`** — complete map state. Sent on initial load and map transitions. Clients discard their current frame and rebuild entirely.

**`DeltaSnapshot`** — only cells that changed since the last snapshot. Clients apply these on top of their current frame. `BuildDelta()` returns `null` if nothing changed — the game loop skips the render call.

### What a snapshot carries

```
Raw IDs only — no resolved art, no sprite paths, no ASCII arrays.
Clients resolve art references from their own asset cache.

CellSnapshot:
  x, y
  LayerStack: [ LayerData, ... ]   bottom → top

LayerData:
  Kind:     Tile | Item | Npc | Player
  ArtId:    string  (cache key clients use for art lookup)
  EntityId: string  (npc_id, item_id — null for tile/player layers)
```

### Layer resolution

```
For each tile position (x, y):

  Layer 0 — Tile (always present)
    ArtId = tile name from tileset definition

  Layer 1 — Item (if ground item present at x_y in world state)
    ArtId = "item_ground_{item_id}" if entity art exists,
            else "item_ground" (generic fallback)

  Layer 2 — NPC (if any NPC positioned at x, y in world state)
    ArtId = npc.entity_art if set,
            else "npc_default"

  Layer 3 — Player (if player_pos matches x, y)
    ArtId = "player"

Text client:   renders LayerStack.Top (highest priority layer present)
Sprite client: iterates full stack bottom → top, composites with alpha
```

### IRenderer interface

```csharp
public interface IRenderer
{
    void RenderFull(FullSnapshot snapshot);   // rebuild entire frame
    void RenderDelta(DeltaSnapshot snapshot); // apply changed cells only
}
```

### Delta tracking

`SnapshotBuilder` keeps a `_previousCells` dictionary keyed by `(mapId, x, y)`. Each frame it rebuilds every cell's `LayerStack` and compares it structurally to the previous frame. Only changed cells appear in the delta. The map ID in the key means a map transition automatically invalidates all cached cells.

---

## 12. YAML Schemas

### items.yaml

```yaml
- id: iron_sword           # string, required, unique
  name: Iron sword         # string, required
  type: weapon             # string, required (weapon|armor|consumable|material)
  description: ...         # string
  flavor: ...              # string — meta/snarky tooltip text
  sprite: path/to/img.png  # string — sprite client asset path
  slot: main_hand          # string — equipment slot
  stackable: false         # bool
  tradeable: true          # bool
  weight: 2.5              # float
  value: 120               # int — base gold value
  skill_requirement:
    skill: melee           # skill id
    level: 1               # minimum level to equip
  craft_requirement:
    skill: blacksmithing   # skill id
    level: 20              # minimum level to craft
  stats:
    attack: 12             # any float stat — fully dynamic
    defense: 0
    speed: 1.0
  use_effect:              # list of effect primitives (consumables only)
    - emit: { event: player.healed, payload: { amount: 25 } }
```

### skills.yaml

```yaml
xp_curve:
  formula: exponential
  base: 100
  exponent: 1.15
level_cap: 99
milestone_levels: [10, 20, 30, 40, 50, 60, 70, 80, 90, 99]

skills:
  - id: blacksmithing        # string, required
    name: Blacksmithing      # string, required
    category: crafting       # string, required (crafting|magic|combat)
    description: ...
    flavor: ...
    icon: skills/icon.png
    xp_source: crafting_action  # crafting_action | combat_kill
    train_locations:
      - smithy               # list of building IDs
    training_cost:           # magic skills only
      material_id: mana_crystal
      amount_per_action: 1
    milestones:
      - level: 20
        label: Journeyman smith
        flavor: ...
        effects:             # list of effect primitives
          - add: { path: player.known_recipes, value: iron_sword }
          - emit: { event: base.worker_unlocked, payload: { worker: apprentice_smith } }
```

### tilesets.yaml

```yaml
- id: overworld             # string, required
  name: Overworld tileset
  tiles:
    - char: "."             # single ASCII character, required
      name: Grass           # string, required
      sprite: tiles/...     # sprite client path
      walkable: true        # bool, required
      color:
        fg: bright_green    # ANSI color name
        bg: black
      ascii: |              # exactly 16 lines of exactly 16 characters
        .,. , . .,. ,  ,
        ...
      on_enter:             # optional effect list
        - emit: { ... }
      on_interact:          # optional effect list
        - emit: { ... }
```

### maps.yaml

```yaml
- id: town_square           # string, required
  name: Town square         # string, required
  tileset: overworld        # tileset id, required
  music: music/theme.ogg
  grid: |                   # ASCII art grid, required
    ################
    #..............#
    ##.####.#######
  legend:
    ".": grass              # simple: character → tileset tile name
    ">A":                   # override: adds instance-specific fields
      tileset_tile: passage_down
      target_map: blacksmith_district
      target_x: 7
      target_y: 1
  npcs:
    - npc_id: innkeeper_harold
      x: 7
      y: 6
  items:
    - item_id: old_notice
      x: 7
      y: 2
      quantity: 1
  default_spawn:
    x: 7
    y: 10
```

### systems/*.yaml

```yaml
on: event.name              # trigger event, required
failure_event: event.failed # emitted on any rule failure (optional)
rules:
  - check: rule_name        # registered rule function name
    source: door            # optional rule params (passed to ctx.RuleParam)
    on_fail:                # effects to run if this rule fails
      - emit: { ... }
config:
  xp_amount: 25             # static values available as $$xp_amount in effects
success_effects:            # effects run if all rules pass
  - emit: { ... }
  - set:  { ... }
```

### entities.yaml

```yaml
- id: player                # string, required — matches LayerData.ArtId
  name: Player character
  color:
    fg: bright_white
    bg: black
  ascii: |                  # exactly 16 lines of exactly 16 characters
    ...
```

---

## 13. Contributor Patterns

### Adding a new content type

1. Create `/content/thing.yaml` with `id` and any fields you need
2. Add `"thing": ["id", "name"]` to `SchemaValidator._requiredFields`
3. Add `content.LoadList("things", "content/thing.yaml")` in `GameBootstrap`
4. Done — use `content.TryGet("things", id)` anywhere

### Adding a new game system

1. Create `/systems/my_system.yaml` with `on`, `rules`, `success_effects`
2. Reference only already-registered rule names
3. `SystemLoader.LoadAll()` picks it up automatically — no C# changes

### Adding a new rule

1. Write a static method in the appropriate rules class in `GameSystems.cs`:
   ```csharp
   public static bool MyNewRule(RuleContext ctx)
   {
       var source = ctx.RuleParam<string>("source");
       var id     = ctx.EventValue($"{source}_id");
       var def    = ctx.GetDefinition(source + "s", id);
       // ... check something, return bool
   }
   ```
2. Register it in `GameBootstrap`:
   ```csharp
   rules.Register("my_new_rule", MyRuleClass.MyNewRule);
   ```
3. Use it in any system YAML:
   ```yaml
   - check: my_new_rule
     source: door
   ```

### Adding a new built-in

Built-ins are for cases where you need to both read state *and* execute effects — something a pure `bool` rule cannot do. Use them sparingly.

```csharp
executor.RegisterBuiltin("my_builtin", args =>
{
    var x = Convert.ToInt32(args["x"]);
    // read state, execute effects, emit events
});
```

Call from YAML via: `- call: { fn: my_builtin, args: { x: "$$target_x" } }`

### Adding a new map area

1. Add a tileset entry in `tilesets.yaml` if you need new tile types
2. Add the map definition in `maps.yaml` with an ASCII grid and legend
3. Add transition tiles in existing maps pointing to your new map's ID
4. Place NPCs and items in the `npcs` and `items` lists
5. No C# changes needed

### Adding a new renderer

1. Implement `IRenderer` in a new class
2. Inject `SnapshotBuilder` from `EngineContext`
3. Call `BuildFull()` on map load, `BuildDelta()` each frame
4. Resolve `LayerData.ArtId` against your own asset cache
5. No engine changes needed

---

## 14. What Stays in C# vs YAML

### Always in C#

- The four effect primitives (`set`, `add`, `emit`, `call`)
- Dot-path walking logic
- Event bus routing
- YAML parsing and schema validation
- Rule function implementations (the actual boolean logic)
- Built-in function implementations
- XP curve math
- ASCII grid parsing
- Delta snapshot computation
- `IRenderer` implementations

### Always in YAML

- All content definitions (items, skills, maps, tilesets, entities, NPCs, quests)
- All game system triggers, rule lists, and success effects
- All milestone effects
- All tile behaviors (on_enter, on_interact)
- All map layouts and area connections
- All NPC and item placements
- All stat values, XP amounts, recipe lists
- All flavor text and meta dialogue

### The grey area — rule params

Rule *names* are in YAML. Rule *logic* is in C#. Rule *parameters* (like `source: door`) are in YAML and passed to the rule via `ctx.RuleParam()`. This is intentional — it lets the same rule function work across different content types without any C# changes.

---

*Document reflects architectural state as of initial engine design phase. Update when new systems, rules, or content types are added.*
