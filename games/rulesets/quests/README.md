# Module: quests v1.0.0

Quest availability, prerequisites, objective tracking, and completion logic.

> Source: [DG batteries catalog](https://app.datagrout.ai/batteries/quests), loaded into your namespace on install.

## Install

```lua
dg:batteries().install("quests", "my-game", function(result)
  print("Installed " .. result.predicate_count .. " predicates")
end)
```

## Exported Predicates

| Predicate | Description |
|---|---|
| `quest_available(Player, Quest)` | Quest is available for Player to accept |
| `quest_in_progress(Player, Quest)` | Player has accepted but not completed Quest |
| `quest_complete(Player, Quest)` | Player has fully completed Quest |
| `next_objective(Player, Quest, Obj)` | Obj is the current unmet objective in order |
| `quest_blocked_by(Player, Quest, Reason)` | Quest unavailable — Reason explains why |
| `can_turn_in(Player, Quest)` | All objectives met, ready to hand in |

## Setup

```lua
-- Register quests in the world
dg:assert("my-game", { type="relation", subject="world", relation="quest_exists", object="slay_dragon" })

-- Add prerequisites
dg:assert("my-game", { type="attribute", entity="slay_dragon", attribute="requires_level", value=10 })
dg:assert("my-game", { type="attribute", entity="slay_dragon", attribute="requires_quest", value="find_sword" })

-- Add objectives (in order)
dg:assert("my-game", { type="attribute", entity="slay_dragon_obj1", attribute="quest",       value="slay_dragon" })
dg:assert("my-game", { type="attribute", entity="slay_dragon_obj1", attribute="order",       value=1 })
dg:assert("my-game", { type="attribute", entity="slay_dragon_obj1", attribute="description", value="Find the dragon's lair" })
dg:assert("my-game", { type="attribute", entity="slay_dragon_obj2", attribute="quest",       value="slay_dragon" })
dg:assert("my-game", { type="attribute", entity="slay_dragon_obj2", attribute="order",       value=2 })
dg:assert("my-game", { type="attribute", entity="slay_dragon_obj2", attribute="description", value="Defeat the dragon" })
```

## Usage

```lua
-- What quests can alice accept right now?
dg:query("my-game", "quest_available(alice, Quest)", function(results)
  for _, r in ipairs(results) do showQuestOffer(r.Quest) end
end)

-- Why can't alice take a quest?
dg:query("my-game", "quest_blocked_by(alice, slay_dragon, Reason)", function(results)
  if results[1] then
    showHint(results[1].Reason)  -- e.g. requires_level(10)
  end
end)

-- Mark an objective complete
local function completeObjective(player, objective)
  local key = player .. "_" .. objective
  dg:assert("my-game", { type="attribute", entity=key, attribute="complete", value=true })
end

-- What should alice do next?
dg:query("my-game", "next_objective(alice, slay_dragon, Obj)", function(results)
  if results[1] then updateQuestTracker(results[1].Obj) end
end)

-- Is alice ready to turn in the quest?
dg:query("my-game", "can_turn_in(alice, slay_dragon)", function(results)
  if #results > 0 then showTurnInPrompt() end
end)
```
