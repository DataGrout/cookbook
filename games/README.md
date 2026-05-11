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

**Queries cost zero tokens.** Answers are computed by the Prolog engine — not an LLM. This means query results are deterministic, instant, and free regardless of how complex the rules are. The LLM (or you) writes the rules once; the engine evaluates them for free on every call. This is sometimes called symbolic AI or neuro-symbolic AI.

## The Game Loop Pattern

```
Game event happens
       |
  assert facts        -- update what's true
       |
  query rules         -- ask what's possible / what should happen
       |
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
| Token cost per query | — | zero |

For simple key/value storage use DataStore. For anything involving conditions, prerequisites, relationships, or derived state, Logic Cells are less code and more flexible.

## Batteries — Pre-built Rule Modules

Instead of writing rules from scratch, install pre-built rule modules from the [logic-batteries](https://github.com/datagrout/logic-batteries) library. Each battery is a tested set of Prolog predicates you can install into any namespace in one call.

```lua
-- Install the inventory system
dg:batteries().install("inventory", "my-game", function(result)
  print("Installed: " .. result.predicate_count .. " predicates")
end)

-- Query immediately — no setup, no schema
dg:query("my-game", "can_carry(alice, Item)", function(results) ... end)
```

### Game batteries

| Battery | What it does |
|---------|-------------|
| `inventory` | Item carrying, weight limits, slot constraints |
| `loot-tables` | Rarity tiers, condition-gated drops |
| `quests` | Prerequisites, progress tracking, branching objectives |
| `combat` | Damage, resistances, status effects, turn order |
| `progression` | XP, level thresholds, stat unlocks |
| `economy` | Crafting, pricing, supply/demand |
| `npc-state` | Relationships, dialogue conditions, factions |
| `faction` | Reputation, allegiances, conflict rules |
| `dialogue` | Dialogue availability, conditions, branching |
| `crafting` | Recipe resolution, material substitution |
| `world` | World state, region access, environmental conditions |
| `puzzle-fsm` | State machines, win conditions, hint generation |
| `dungeon` | Navigation, room connectivity, path finding |
| `ai-director` | Pacing, difficulty scaling, encounter selection |
| `permissions` | Access control, role-based gates |

Usage guides for the core modules: [inventory](./rulesets/inventory/) — [loot-tables](./rulesets/loot-tables/) — [quests](./rulesets/quests/)

### Business batteries for simulation games

The full catalog also includes a business rules suite — useful if you're building economic or simulation games:

| Battery | What it does |
|---------|-------------|
| `pricing-rules` | Dynamic pricing, discounts, margin floors |
| `inventory-mgmt` | Stock levels, reorder triggers, allocation |
| `approval-chains` | Multi-step approval workflows with conditions |
| `scheduling` | Resource booking, conflict detection |
| `loyalty` | Points, tiers, reward eligibility |
| `lead-scoring` | Funnel scoring, qualification rules |
| `invoice-rules` | Billing logic, payment terms, overdue detection |
| `compliance` | Policy enforcement, audit trail rules |

A business simulator — trading company, logistics game, tycoon — can install `pricing-rules`, `inventory-mgmt`, and `scheduling` and get a working economic engine backed by symbolic reasoning, zero tokens per tick.

Browse the full catalog: [github.com/datagrout/logic-batteries](https://github.com/datagrout/logic-batteries)

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
