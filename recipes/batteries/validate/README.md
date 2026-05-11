# Recipe: Validating a Battery Installation

> **Difficulty:** Beginner  
> **Credits:** 0 (validation is read-only)  
> **Time:** ~5 minutes

## What it does

After installing a battery you sometimes get empty results from queries. `batteries.validate` runs a diagnostic pass, one open query per predicate, and tells you whether each one is:

- **pass** — firing and returning results
- **no_data** — loaded but no matching LC facts exist yet (you need to assert)
- **not_loaded** — predicate is unknown to Prolog (battery not installed)
- **error** — unexpected failure

It also returns concrete `lc.assert` snippets showing exactly which facts are missing, and an optional signed CTC you can archive as proof of validation.

## Tools used

| Tool | Purpose | Credits |
|------|---------|---------|
| `batteries.install_many` | Load battery predicates into namespace | 0 |
| `batteries.validate` | Diagnose predicate readiness | 0 |
| `logic.assert` | Assert the missing facts | 1 |

## The Recipe

### Step 1: Install and immediately validate

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

# Install
client.perform("data-grout@1/batteries.install_many@1", {
    "ids": ["inventory"],
    "namespace": "my-game"
})

# Validate right away: all predicates will show no_data
# because we haven't asserted any facts yet
result = client.perform("data-grout@1/batteries.validate@1", {
    "id": "inventory",
    "namespace": "my-game"
})

print(result["overall"])        # "fail"
print(result["pass_count"])     # 0
print(result["fail_count"])     # 3 (or however many predicates)
```

### Step 2: Read the diagnosis

Each predicate entry tells you exactly what's wrong:

```python
for p in result["predicates"]:
    print(f"{p['predicate']:20s}  {p['status']}")
    if p["missing_facts"]:
        for hint in p["missing_facts"]:
            print(f"    -> {hint}")
```

Output:
```
can_carry             no_data
    -> lc.assert: {"type": "attribute", "entity": "<entity>", "attribute": "max_weight", "value": "<value>"}
    -> -- e.g. to enable can_carry, assert the attribute fact that drives it
inventory_full        no_data
    -> lc.assert: {"type": "attribute", "entity": "<entity>", "attribute": "carrying", "value": "<value>"}
carrying_weight       no_data
```

### Step 3: Assert the missing facts

```python
client.perform("data-grout@1/logic.assert@1", {
    "namespace": "my-game",
    "facts": [
        {"type": "entity", "name": "player1", "attributes": {
            "max_weight": 100,
            "max_slots": 10
        }},
        {"type": "attribute", "entity": "player1", "attribute": "carrying", "value": "sword"},
        {"type": "attribute", "entity": "sword", "attribute": "weight", "value": 12},
        {"type": "attribute", "entity": "sword", "attribute": "slot",   "value": "right_hand"},
    ]
})
```

### Step 4: Re-validate to confirm

```python
result = client.perform("data-grout@1/batteries.validate@1", {
    "id": "inventory",
    "namespace": "my-game"
})

print(result["overall"])    # "pass"
print(result["pass_count"]) # 3
```

## Reading `predicate_docs` for guidance

`batteries.describe` now returns `predicate_docs`, per-predicate documentation extracted from the battery source. Use it when the `validate` hints aren't specific enough:

```python
docs = client.perform("data-grout@1/batteries.describe@1", {
    "id": "inventory"
})

for p in docs["predicate_docs"]:
    print(f"{p['signature']:25s}  {p['doc']}")
```

Output:
```
can_carry/2               can_carry(Player, Item) is true when the player's current carrying weight
                          plus the item weight is under max_weight, and the required slot is free.
carrying_weight/2         carrying_weight(Player, W) binds W to the total weight of all items
                          the player is currently carrying.
```

## Produce a validation certificate

Pass `produce_ctc: true` to mint a signed artifact recording the outcome. Useful for CI gates or audit trails:

```python
result = client.perform("data-grout@1/batteries.validate@1", {
    "id": "inventory",
    "namespace": "my-game",
    "produce_ctc": True
})

ctc = result["ctc"]
print(ctc["id"])            # "ctc-battery-validate-Xk4..."
print(ctc["viewer_url"])    # Link to the CTC viewer
print(ctc["assurances"])    # {"all_predicates_pass": true, "pass_count": 3, ...}
```

## Common patterns

**Validate after every data load:**
```python
def load_player_facts(player_id, facts):
    client.perform("data-grout@1/logic.assert@1", {"namespace": "my-game", "facts": facts})
    result = client.perform("data-grout@1/batteries.validate@1",
                            {"id": "inventory", "namespace": "my-game"})
    if result["overall"] != "pass":
        failing = [p["predicate"] for p in result["predicates"] if p["status"] != "pass"]
        raise RuntimeError(f"Battery not ready after load: {failing}")
```

**Validate in CI before shipping a namespace snapshot:**
```python
for battery_id in ["inventory", "quests", "loot-tables"]:
    r = client.perform("data-grout@1/batteries.validate@1", {
        "id": battery_id,
        "namespace": "staging",
        "produce_ctc": True
    })
    assert r["overall"] == "pass", f"{battery_id}: {r['recommendations']}"
    print(f"✓ {battery_id} — {r['pass_count']}/{r['predicate_count']} predicates pass")
```

## `overall` values

| Value | Meaning |
|-------|---------|
| `pass` | All predicates fired |
| `partial` | Some pass, some fail |
| `fail` | No predicates firing (all no_data or errors) |
| `not_installed` | Battery predicates not loaded — run install_many first |

## See also

- [getting-started](../getting-started/) — install and first queries
- [composing-batteries](../composing-batteries/) — multi-battery namespaces
- [assert-and-query](../../logic/assert-and-query/) — LC fact shapes
