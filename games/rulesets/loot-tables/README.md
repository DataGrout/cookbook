# Module: loot-tables v1.0.0

Rarity tiers, condition-gated drops, and drop chance calculation for any game with item drops.

> Source: [DG batteries catalog](https://app.datagrout.ai/batteries/loot-tables), loaded into your namespace on install.

## Install

```lua
dg:batteries().install("loot-tables", "my-game", function(result)
  print("Installed " .. result.predicate_count .. " predicates")
end)
```

## Exported Predicates

| Predicate | Description |
|---|---|
| `drops(Source, Item)` | Item can drop from Source under any conditions |
| `drops_at(Source, Item, Conditions)` | Item drops from Source when Conditions are met |
| `eligible_loot(Source, Item)` | Item is eligible to drop right now given current world state |
| `rarity_tier(Item, Tier)` | Tier is `common`/`uncommon`/`rare`/`epic`/`legendary` |
| `condition_met(Condition, Context)` | Condition is satisfied in Context |
| `loot_chance(Source, Item, Pct)` | Pct is drop chance 0–100 |

## Default Drop Chances by Rarity

| Tier | Chance |
|---|---|
| common | 70% |
| uncommon | 30% |
| rare | 10% |
| epic | 3% |
| legendary | 1% |

Override per-item with an explicit `drop_chance` attribute.

## Setup

```lua
-- Register what a source can drop
dg:assert("my-game", { type="relation", subject="cave_chest", relation="can_drop", object="gold_coin" })
dg:assert("my-game", { type="relation", subject="cave_chest", relation="can_drop", object="rare_gem" })

-- Set rarity (default is common)
dg:assert("my-game", { type="attribute", entity="rare_gem", attribute="rarity", value="rare" })

-- Conditional drop: rare_fish only drops at night in rain
dg:assert("my-game", { type="relation", subject="lake", relation="can_drop", object="rare_fish" })
dg:assert("my-game", { type="attribute", entity="rare_fish", attribute="rarity", value="uncommon" })
dg:assert("my-game", { type="attribute", entity="rare_fish", attribute="loot_conditions",
  value="[time(night), weather(rain)]" })

-- Set world conditions at runtime
dg:assert("my-game", { type="attribute", entity="world", attribute="time_of_day", value="night" })
dg:assert("my-game", { type="attribute", entity="world", attribute="weather", value="rain" })
```

## Usage

```lua
-- What can drop from this chest right now?
dg:query("my-game", "eligible_loot(cave_chest, Item)", function(results)
  for _, r in ipairs(results) do
    addToDropPool(r.Item)
  end
end)

-- What is the drop chance for a specific item?
dg:query("my-game", "loot_chance(cave_chest, rare_gem, Chance)", function(results)
  if results[1] then
    print("Drop chance: " .. results[1].Chance .. "%")
  end
end)

-- Roll drops on kill using eligible_loot + loot_chance
local function rollDrops(source)
  dg:query("my-game", "eligible_loot(" .. source .. ", Item), loot_chance(" .. source .. ", Item, Chance)",
    function(results)
      for _, r in ipairs(results) do
        if math.random(100) <= tonumber(r.Chance) then
          spawnItem(r.Item)
        end
      end
    end)
end
```

## Conditions

The following condition types are built in:

| Condition | Fact to assert |
|---|---|
| `time(night)` | `attribute(world, time_of_day, night)` |
| `weather(rain)` | `attribute(world, weather, rain)` |
| `moon(full)` | `attribute(world, moon_phase, full)` |
| `player_level_gte(10)` | `attribute(Player, level, N)` where N ≥ 10 |
| `player_has(sword)` | `relation(Player, has_item, sword)` |
| `always` | No fact needed — always true |
