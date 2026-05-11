# Recipe: Getting Started with Batteries

> **Difficulty:** Beginner  
> **Credits:** 0 (install is free; queries use your LC credits)  
> **Time:** ~5 minutes

## What it does

Batteries are pre-built Prolog rule modules that load reasoning engines directly into a logic cell namespace. Install a battery and its predicates become immediately queryable alongside your own asserted facts, no schema setup, no boilerplate.

The pattern:
1. **Search**, find a battery for your domain
2. **Describe**, understand its predicates and what facts to assert
3. **Install**, load it into an LC namespace
4. **Assert**, push your domain facts
5. **Query**, use the battery's predicates against your data

## Browsing the catalog

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

# Browse all batteries
all_batteries = client.perform("data-grout@1/batteries.search@1", {})
for b in all_batteries["results"]:
    print(f"{b['id']:20s} {b['title']}")

# Filter by category: reasoning, games, business
reasoning = client.perform("data-grout@1/batteries.search@1", {
    "category": "reasoning"
})

# Search by keyword
inv = client.perform("data-grout@1/batteries.search@1", {
    "query": "inventory"
})
```

## Describe a battery before installing

```python
docs = client.perform("data-grout@1/batteries.describe@1", {
    "id": "inventory",
    "include_rules": False   # set True to see the raw Prolog
})

print(docs["predicates"])       # ['can_carry/2', 'inventory_full/1', ...]
print(docs["fact_schema"])      # what LC facts to assert for each predicate
print(docs["example_queries"])  # ready-to-use Prolog queries
print(docs["composition_hints"])# other batteries that pair well with this one
```

`fact_schema` tells you exactly which facts to assert. For the `inventory` battery:
```python
# can_carry(Player, Item) fires when:
{"type": "attribute", "entity": "player1", "attribute": "max_weight", "value": 100}
{"type": "attribute", "entity": "sword",   "attribute": "weight",     "value": 12}
{"type": "attribute", "entity": "sword",   "attribute": "slot",       "value": "right_hand"}
```

## Install and use: inventory example

```python
# Install the battery into a namespace
install = client.perform("data-grout@1/batteries.install_many@1", {
    "ids": ["inventory"],
    "namespace": "my-game"
})
print(install["installed_count"])   # 1

# Assert player and item facts
client.perform("data-grout@1/logic.assert@1", {
    "namespace": "my-game",
    "facts": [
        # Player capacity
        {"type": "entity", "name": "player1", "attributes": {
            "max_weight": 100,
            "max_slots": 10
        }},
        # Items in inventory
        {"type": "attribute", "entity": "player1", "attribute": "carrying", "value": "sword"},
        {"type": "attribute", "entity": "player1", "attribute": "carrying", "value": "shield"},
        # Item properties
        {"type": "attribute", "entity": "sword",  "attribute": "weight", "value": 12},
        {"type": "attribute", "entity": "sword",  "attribute": "slot",   "value": "right_hand"},
        {"type": "attribute", "entity": "shield", "attribute": "weight", "value": 18},
        {"type": "attribute", "entity": "shield", "attribute": "slot",   "value": "left_hand"},
        # New item the player wants to pick up
        {"type": "attribute", "entity": "heavy_axe", "attribute": "weight", "value": 35},
        {"type": "attribute", "entity": "heavy_axe", "attribute": "slot",   "value": "right_hand"},
    ]
})

# Query using the battery's predicates
result = client.perform("data-grout@1/logic.query@1", {
    "namespace": "my-game",
    "query": "can_carry(player1, heavy_axe)"
})
print(result)   # True or False

weight = client.perform("data-grout@1/logic.query@1", {
    "namespace": "my-game",
    "query": "carrying_weight(player1, W)"
})
print(weight)   # W = 30 (sword + shield)
```

## Check what's installed

```python
installed = client.perform("data-grout@1/batteries.installed@1", {
    "namespace": "my-game"
})
for b in installed["batteries"]:
    print(f"{b['id']} v{b['version']}")
```

## Remove a battery

Removing a battery retracts its predicate rules. Your asserted facts stay intact, only the battery's own logic is removed.

```python
client.perform("data-grout@1/batteries.remove@1", {
    "id": "inventory",
    "namespace": "my-game"
})
```

## What makes batteries useful

Without a battery:
```python
# You'd write Prolog rules by hand and assert them as constraints
client.perform("data-grout@1/logic.constrain@1", {
    "rule": "A player can carry an item if their current weight plus the item's weight is under their max_weight",
    "namespace": "my-game"
})
```

With a battery:
```python
# One install call loads battle-tested rules covering weight, slots, stacking, and more
client.perform("data-grout@1/batteries.install_many@1", {"ids": ["inventory"], "namespace": "my-game"})
```

The battery handles edge cases you'd otherwise discover the hard way (slot conflicts, weight rounding, stacking limits).

## See also

- [composing-batteries](../composing-batteries/) — combine inventory + quests + loot-tables in one namespace
- [assert-and-query](../../logic/assert-and-query/) — LC basics if you're new to logic cells
- [games/rulesets](../../../games/) — the Prolog source for each battery
