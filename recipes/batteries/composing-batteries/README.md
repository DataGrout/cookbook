# Recipe: Composing Multiple Batteries

> **Difficulty:** Intermediate  
> **Credits:** 0 (install is free)  
> **Time:** ~10 minutes

## What it does

Install multiple batteries into a single namespace and query across their predicates. Because all batteries share the same fact space, predicates from different batteries compose naturally through the facts you assert, `inventory` knows about items, `quests` knows about objectives, and `loot-tables` knows about drop chances. One assertion feeds all three.

## The pattern

```
assert facts once -> batteries compose through shared entities -> query any predicate
```

## Installing multiple batteries at once

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

result = client.perform("data-grout@1/batteries.install_many@1", {
    "ids": ["inventory", "quests", "loot-tables"],
    "namespace": "rpg-session"
})

print(result["installed_count"])         # 3
print(result["already_installed_count"]) # 0 on first run; skips re-installs idempotently
```

## Asserting shared facts

Facts assert once and are visible to all installed batteries:

```python
client.perform("data-grout@1/logic.assert@1", {
    "namespace": "rpg-session",
    "facts": [
        # Player state
        {"type": "entity", "name": "player1", "attributes": {
            "max_weight": 100,
            "max_slots": 10,
            "level": 5,
            "gold": 250
        }},
        # Current inventory
        {"type": "attribute", "entity": "player1", "attribute": "carrying", "value": "sword"},
        {"type": "attribute", "entity": "player1", "attribute": "carrying", "value": "potion"},
        # Item properties (used by inventory + loot-tables)
        {"type": "attribute", "entity": "sword",    "attribute": "weight",     "value": 12},
        {"type": "attribute", "entity": "sword",    "attribute": "slot",       "value": "right_hand"},
        {"type": "attribute", "entity": "potion",   "attribute": "weight",     "value": 1},
        {"type": "attribute", "entity": "potion",   "attribute": "stackable",  "value": True},
        # Loot table entry
        {"type": "attribute", "entity": "dragon_boss", "attribute": "drops", "value": "legendary_axe"},
        {"type": "attribute", "entity": "legendary_axe","attribute": "rarity", "value": "legendary"},
        {"type": "attribute", "entity": "legendary_axe","attribute": "weight", "value": 40},
        {"type": "attribute", "entity": "legendary_axe","attribute": "drop_chance", "value": 0.05},
        # Quest state
        {"type": "attribute", "entity": "player1", "attribute": "quest_active", "value": "slay_dragon"},
        {"type": "attribute", "entity": "slay_dragon", "attribute": "requires_item", "value": "sword"},
        {"type": "attribute", "entity": "slay_dragon", "attribute": "objective_complete",
         "value": "found_lair"},
        {"type": "attribute", "entity": "slay_dragon", "attribute": "objective_pending",
         "value": "defeat_boss"},
    ]
})
```

## Cross-battery queries

Now all three batteries' predicates work against the same facts:

```python
# inventory battery: can the player carry the legendary axe?
can_carry = client.perform("data-grout@1/logic.query@1", {
    "namespace": "rpg-session",
    "query": "can_carry(player1, legendary_axe)"
})
# -> False (weight 12+1+40 = 53, but slot conflict with sword)

# loot-tables battery: is it eligible to drop?
eligible = client.perform("data-grout@1/logic.query@1", {
    "namespace": "rpg-session",
    "query": "eligible_loot(dragon_boss, legendary_axe)"
})
# -> True

# quests battery: what's the next objective?
next_obj = client.perform("data-grout@1/logic.query@1", {
    "namespace": "rpg-session",
    "query": "next_objective(player1, slay_dragon, Obj)"
})
# -> Obj = defeat_boss

# Cross-battery: can they complete the quest AND carry the reward?
# Compose with two queries and Python logic
quest_ready = client.perform("data-grout@1/logic.query@1", {
    "namespace": "rpg-session",
    "query": "quest_available(player1, slay_dragon)"
})
can_loot = client.perform("data-grout@1/logic.query@1", {
    "namespace": "rpg-session",
    "query": "can_carry(player1, legendary_axe)"
})

if quest_ready and not can_loot:
    print("Quest is available but player needs to drop sword before looting the axe")
```

## Natural-language query with logic.query + analyze

```python
# Ask a semantic question that the LLM resolves to Prolog queries
result = client.perform("data-grout@1/logic.query@1", {
    "namespace": "rpg-session",
    "goal": "Is player1 ready to fight the dragon, and can they carry the legendary axe if it drops?",
    "mode": "semantic"   # LLM generates the Prolog queries
})
print(result["answer"])
print(result["queries_used"])  # see what Prolog was generated
```

## Upgrade a battery without losing facts

Batteries are versioned. Upgrading retracts the old rules and loads the new ones. Your asserted facts are never touched.

```python
# Re-install to upgrade to latest version
client.perform("data-grout@1/batteries.install_many@1", {
    "ids": ["inventory"],
    "namespace": "rpg-session"
})
# already_installed_count=1 if same version; installed_count=1 if upgraded
```

## Adding a custom rule alongside batteries

Your own `logic.constrain` rules coexist with battery predicates:

```python
# Add a custom rule: legendary items can only be carried by level 5+ players
client.perform("data-grout@1/logic.constrain@1", {
    "rule": "A player can only carry legendary items if their level is 5 or above",
    "namespace": "rpg-session"
})

# Now can_carry(player1, legendary_axe) incorporates both the inventory
# battery's weight/slot rules AND your level restriction
```

## Namespace isolation

Each game session, environment, or tenant gets its own namespace. Batteries and facts are fully isolated:

```python
# Player A's session
client.perform("data-grout@1/batteries.install_many@1", {
    "ids": ["inventory", "quests"],
    "namespace": "session-player-A"
})

# Player B's session: same batteries, completely separate facts
client.perform("data-grout@1/batteries.install_many@1", {
    "ids": ["inventory", "quests"],
    "namespace": "session-player-B"
})
```

## See also

- [getting-started](../getting-started/) — install and query a single battery
- [assert-and-query](../../logic/assert-and-query/) — LC facts and Prolog queries
- [games/rulesets](../../../games/) — Prolog source for inventory, quests, and loot-tables
