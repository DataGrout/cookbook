# Games: Logic Cells for Game Developers

This section is for game developers using [Tether](https://github.com/datagrout/tether). It assumes no prior knowledge of Prolog or AI.

## What is a Logic Cell?

A Logic Cell (LC) is a named knowledge base living in the cloud. Think of it as a database for your game's rules and state, but instead of SQL rows, you store **facts** and **rules**, and instead of SELECT queries you ask questions in plain logic.

```
namespace "my-fishing-game"
├── facts    , things that are true right now
├── rules    , derived truths computed from facts
└── queries  , questions you ask at runtime
```

Your game asserts facts as things happen, defines rules once, and queries the result. The LC figures out the answers.

## Facts

A fact is a single true statement about your game world.

```lua
-- A player exists
dg:assert("my-game", { type = "entity", name = "alice" })

-- Alice is at level 5
dg:assert("my-game", { type = "attribute", entity = "alice", attribute = "level", value = 5 })

-- Alice has a sword
dg:assert("my-game", { type = "relation", subject = "alice", relation = "has_item", object = "iron_sword" })
```

Facts persist across sessions. Assert them when your game state changes.

## Rules

A rule is a derived truth computed from facts. You define rules once, the LC evaluates them on every query.

```lua
-- Rule: a player can equip an item if they meet the level requirement
dg:assert("my-game", {
  type = "rule",
  head = "can_equip(Player, Item)",
  body = "attribute(Player, level, L), attribute(Item, req_level, R), L >= R"
})
```

You never call rules directly, you query them the same way you query facts.

## Queries

A query asks a question. The LC pattern-matches against all facts and rules and returns every answer.

```lua
-- What items can alice equip right now?
dg:query("my-game", "can_equip(alice, Item)", function(results)
  for _, r in ipairs(results) do
    print(r.Item)  -- "iron_sword", "leather_boots", ...
  end
end)
```

Variables are uppercase. The LC fills them in.

## The Game Loop Pattern

```
Game event happens
       ↓
  assert facts        -- update what's true
       ↓
  query rules         -- ask what's possible / what should happen
       ↓
  game reacts         -- render the answer
```

Example: player picks up an item:
1. `assert` -> `{ relation: "has_item", subject: "alice", object: "health_potion" }`
2. `query` -> `"can_use(alice, Item)"`, rules figure out usable items from current state
3. Game enables the "Use" button for health_potion

## Why Not Just Use DataStore?

DataStore stores values you look up by key. Logic Cells store relationships you reason over.

| | DataStore | Logic Cell |
|---|---|---|
| Lookup by key | ✓ | ✓ |
| "What items can this player use?" | write code | write one rule |
| "What quests are available given current state?" | write code | write one rule |
| Rule changes apply immediately | ✗ | ✓ |
| LLM can write the rules for you | ✗ | ✓ |

For simple key/value storage use DataStore. For anything involving conditions, prerequisites, relationships, or derived state, Logic Cells are less code and more flexible.

## Module System

Instead of writing rules from scratch, install pre-built rule modules:

```lua
-- Browse available modules
dg:batteries().list(function(catalog) ... end)

-- Install the inventory system
dg:batteries().install("inventory", "my-game", function(result)
  print("Inventory rules installed: " .. result.predicate_count .. " predicates")
end)

-- Query immediately
dg:query("my-game", "can_carry(alice, Item)", function(results) ... end)
```

See [rulesets/](./rulesets/) for all available modules.

## Rulesets

The pages below are usage guides, they show what facts to assert and what queries to run. The actual Prolog source for each module lives in the [DG batteries catalog](https://app.datagrout.ai/batteries) and is loaded into your namespace on install. You never need to manage the source yourself.

| Module | Usage guide | What it does |
|--------|-------------|--------------|
| `inventory` | [usage ->](./rulesets/inventory/) | Item carrying, weight limits, slot constraints |
| `loot-tables` | [usage ->](./rulesets/loot-tables/) | Rarity tiers, condition-gated drops |
| `quests` | [usage ->](./rulesets/quests/) | Prerequisites, progress tracking, branching |
| `combat` | catalog | Damage, resistances, status effects, turn order |
| `progression` | catalog | XP, level thresholds, stat unlocks |
| `economy` | catalog | Crafting, pricing, supply/demand |
| `npc-state` | catalog | Relationships, dialogue conditions, factions |
| `puzzle-fsm` | catalog | State machines, win conditions, hints |

## Making Rules Visible to AI

When you install modules and call `dg:orchestrate`, the orchestrator automatically discovers what rule predicates exist in your namespace. A dungeon master agent that knows `can_equip/2`, `quest_available/2`, and `npc_friendly/2` exists will use those predicates when deciding what NPCs say and what quests to offer, without you writing any AI logic.

```lua
-- Install modules
dg:batteries().install("quests", "my-game", function() end)
dg:batteries().install("npc-state", "my-game", function() end)

-- The orchestrator knows about quest_available and npc_friendly automatically
dg:orchestrate("dungeon-master", {
  input = "player entered the tavern",
  namespace = "my-game",
}, function(result)
  -- result contains NPC dialogue, quest offers, etc.
  -- all grounded in your actual game state via LC queries
end)
```
