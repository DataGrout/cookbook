# Module: inventory v1.0.0

Item carrying, weight limits, and slot constraints for any game with an inventory system.

> Source: [DG batteries catalog](https://app.datagrout.ai/batteries/inventory), loaded into your namespace on install.

## Install

```lua
dg:batteries().install("inventory", "my-game", function(result)
  print("Installed " .. result.predicate_count .. " predicates")
end)
```

## Exported Predicates

| Predicate | Description |
|---|---|
| `can_carry(Player, Item)` | True if Player can pick up Item without exceeding weight or slot limits |
| `inventory_full(Player)` | True if Player has no carry capacity left |
| `carrying_weight(Player, W)` | W is the total weight Player is currently carrying |
| `item_count(Player, N)` | N is the number of items Player is carrying |
| `has_item(Player, Item)` | True if Player currently has Item |
| `item_in_slot(Player, Slot, Item)` | Item is equipped in Slot for Player |
| `slot_available(Player, Slot)` | Slot is empty and available for Player |

## Defaults

Everything works out of the box. Override by asserting attribute facts:

```lua
-- Increase Alice's carry weight to 80 (default: 50)
dg:assert("my-game", { type="attribute", entity="alice", attribute="max_carry_weight", value=80 })

-- Set inventory slots to 30 (default: 20)
dg:assert("my-game", { type="attribute", entity="alice", attribute="max_slots", value=30 })

-- Set item weight (default: 1)
dg:assert("my-game", { type="attribute", entity="iron_sword", attribute="weight", value=8 })
```

## Usage

```lua
-- Player picks up an item
local function onPickup(player, item)
  dg:query("my-game", "can_carry(" .. player .. ", " .. item .. ")", function(results)
    if #results > 0 then
      -- Grant item in game
      giveItemToPlayer(player, item)
      -- Update the LC
      dg:assert("my-game", { type="relation", subject=player, relation="has_item", object=item })
    else
      notify(player, "Your inventory is full.")
    end
  end)
end

-- What can the player carry right now?
dg:query("my-game", "can_carry(alice, Item)", function(results)
  for _, r in ipairs(results) do
    highlightItem(r.Item)
  end
end)

-- Is Alice's inventory full?
dg:query("my-game", "inventory_full(alice)", function(results)
  if #results > 0 then showFullInventoryWarning() end
end)
```

## Composing with Other Modules

Works naturally with `combat` (weapon equip checks), `quests` (item prerequisites), and `economy` (tradeable items).

```lua
-- Combined: can alice equip the sword AND does she have it?
dg:query("my-game", "has_item(alice, iron_sword), can_equip(alice, iron_sword)", cb)
```
