# Recipe: Custom Prolog Rules with logic.constrain

> **Difficulty:** Intermediate  
> **Credits:** 1 per constrain call  
> **Time:** ~10 minutes

## What it does

`logic.constrain` loads a custom Prolog rule into an LC namespace. Once loaded, the rule becomes a first-class predicate alongside your asserted facts and any installed batteries.

Use it when:
- Batteries don't have exactly the logic you need
- You want to encode domain-specific business rules as verifiable Prolog
- You want to derive facts from existing facts without asserting them explicitly

## Tools used

| Tool | Purpose | Credits |
|------|---------|---------|
| `logic.constrain` | Load a Prolog rule into a namespace | 1 |
| `logic.assert` | Assert facts the rule will reason over | 1 |
| `logic.query` | Query derived facts from the rule | 0 (cached) |
| `logic.reflect` | Inspect what rules and facts are loaded | 0 |

## The Recipe

### Step 1: Assert your base facts

Rules derive new facts from existing ones. Assert the base layer first:

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

client.perform("data-grout@1/logic.assert@1", {
    "namespace": "hr",
    "facts": [
        {"type": "attribute", "entity": "alice", "attribute": "department", "value": "engineering"},
        {"type": "attribute", "entity": "alice", "attribute": "level",      "value": "senior"},
        {"type": "attribute", "entity": "alice", "attribute": "tenure_years", "value": 5},

        {"type": "attribute", "entity": "bob",   "attribute": "department", "value": "engineering"},
        {"type": "attribute", "entity": "bob",   "attribute": "level",      "value": "junior"},
        {"type": "attribute", "entity": "bob",   "attribute": "tenure_years", "value": 1},

        {"type": "attribute", "entity": "carol", "attribute": "department", "value": "sales"},
        {"type": "attribute", "entity": "carol", "attribute": "level",      "value": "senior"},
        {"type": "attribute", "entity": "carol", "attribute": "tenure_years", "value": 7},
    ]
})
```

### Step 2: Load a derived rule

Pass a natural language description of the rule. The LC translates it to Prolog internally:

```python
client.perform("data-grout@1/logic.constrain@1", {
    "namespace": "hr",
    "rule": "An employee is eligible for promotion if they are at junior level and have 2 or more years of tenure"
})
```

Or pass raw Prolog for exact control:

```python
client.perform("data-grout@1/logic.constrain@1", {
    "namespace": "hr",
    "rule": "promotion_eligible(E) :- attribute(E, level, junior), attribute(E, tenure_years, T), T >= 2.",
    "raw": True
})
```

### Step 3: Query the derived predicate

```python
eligible = client.perform("data-grout@1/logic.query@1", {
    "namespace": "hr",
    "query": "promotion_eligible(E)"
})
# -> [{"E": "bob"}]  (alice is senior, carol is sales: only bob matches)
```

### Step 4: Inspect loaded rules

`logic.reflect` shows everything in the namespace, facts and constraints:

```python
state = client.perform("data-grout@1/logic.reflect@1", {
    "namespace": "hr"
})
print(state["constraints"])   # list of loaded rules
print(state["fact_count"])    # number of asserted facts
```

## Pattern: Business rule encoding

Rules make implicit business logic explicit and queryable:

```python
# Encode tiered discount logic
rules = [
    "A customer qualifies for a 10% discount if their total_spend exceeds 1000",
    "A customer qualifies for a 20% discount if their total_spend exceeds 5000 and they have been a customer for more than 2 years",
    "A customer is a vip if they qualify for a 20% discount",
]

for rule in rules:
    client.perform("data-grout@1/logic.constrain@1", {
        "namespace": "crm",
        "rule": rule
    })

# Now query VIPs
vips = client.perform("data-grout@1/logic.query@1", {
    "namespace": "crm",
    "query": "vip(Customer)"
})
```

## Pattern: Transitive relationships

Rules excel at reasoning over graphs:

```python
# Assert reporting structure
client.perform("data-grout@1/logic.assert@1", {
    "namespace": "org",
    "facts": [
        {"type": "relation", "subject": "alice",  "relation": "reports_to", "object": "eve"},
        {"type": "relation", "subject": "bob",    "relation": "reports_to", "object": "alice"},
        {"type": "relation", "subject": "charlie","relation": "reports_to", "object": "alice"},
    ]
})

# Define transitive "managed_by" rule
client.perform("data-grout@1/logic.constrain@1", {
    "namespace": "org",
    "rule": """
        managed_by(E, Manager) :- relation(E, reports_to, Manager).
        managed_by(E, Manager) :- relation(E, reports_to, Mid), managed_by(Mid, Manager).
    """,
    "raw": True
})

# Find everyone ultimately managed by Eve
team = client.perform("data-grout@1/logic.query@1", {
    "namespace": "org",
    "query": "managed_by(E, eve)"
})
# -> [{"E": "alice"}, {"E": "bob"}, {"E": "charlie"}]
```

## Pattern: Composing with batteries

Custom rules and battery predicates share the same namespace. They compose freely:

```python
# Install the inventory battery
client.perform("data-grout@1/batteries.install_many@1", {
    "ids": ["inventory"],
    "namespace": "rpg"
})

# Add a custom rule: legendary items need level 10+
client.perform("data-grout@1/logic.constrain@1", {
    "namespace": "rpg",
    "rule": """
        can_equip_legendary(Player, Item) :-
            attribute(Item, rarity, legendary),
            attribute(Player, level, L),
            L >= 10,
            can_carry(Player, Item).
    """,
    "raw": True
})

# The new predicate composes with battery's can_carry
result = client.perform("data-grout@1/logic.query@1", {
    "namespace": "rpg",
    "query": "can_equip_legendary(player1, legendary_sword)"
})
```

## Raw Prolog vs natural language

| Mode | When to use |
|------|-------------|
| Natural language | Quick rules, simple conditions, exploring |
| `raw: True` | Recursive rules, multi-clause predicates, exact arity control |

For recursive rules (transitive closures, graph traversal), always use raw Prolog, the NL translator may not generate the correct recursive form.

## Rule naming

Prolog predicates are named by their head. Avoid shadowing built-in predicates (`member`, `append`, `length`, etc.) and LC reserved predicates (`lc_attribute`, `lc_relation`, `lc_entity`):

```python
# Good: domain-specific names
"promotion_eligible(E) :- ..."
"vip_customer(C) :- ..."
"managed_by(E, M) :- ..."

# Risky: too generic
"valid(X) :- ..."    # may conflict with other rules
"check(X) :- ..."    # unclear semantics
```

## Retract a rule

Rules can be removed without affecting asserted facts:

```python
client.perform("data-grout@1/logic.forget@1", {
    "namespace": "hr",
    "predicate": "promotion_eligible"
})
```

## See also

- [assert-and-query](../assert-and-query/) — LC fact shapes and query patterns
- [batteries/composing-batteries](../../batteries/composing-batteries/) — batteries + custom rules in one namespace
- [batteries/getting-started](../../batteries/getting-started/) — install pre-built rule modules
- [forensic/causal-chain-tracing](../../forensic/causal-chain-tracing/) — transitive causation rules
