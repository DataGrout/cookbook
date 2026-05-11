# Recipe: Assert and Query

> **Difficulty:** Beginner  
> **Credits:** 1–2  
> **Time:** ~5 minutes

## What it does

Assert structured facts into a Logic Cell namespace, then query them with Prolog. Facts persist across sessions and tool calls, assert once, query from anywhere.

## Tools used

| Tool | Credits | Purpose |
|------|---------|---------|
| `logic.assert` | 1 | Write facts to a namespace |
| `logic.query` | 0 | Prolog query over stored facts |
| `logic.worlds` | 0 | List active namespaces |

## Fact types

| type | required fields | use for |
|------|----------------|---------|
| `entity` | `name` | Named things |
| `attribute` | `entity`, `attribute`, `value` | Properties of entities |
| `relation` | `subject`, `relation`, `object` | Relationships between entities |
| `context` | `key`, `value` | Session or task context |
| `rule` | `head`, `body` | Derived facts (Prolog clauses) |

## Asserting facts

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

client.perform("data-grout@1/logic.assert@1", {
    "namespace": "geography",
    "facts": [
        # Entities
        {"type": "entity", "name": "paris"},
        {"type": "entity", "name": "france"},
        # Attributes
        {"type": "attribute", "entity": "paris", "attribute": "population", "value": 2161000},
        {"type": "attribute", "entity": "paris", "attribute": "is_capital",  "value": True},
        # Relations
        {"type": "relation", "subject": "paris", "relation": "capital_of", "object": "france"},
        {"type": "relation", "subject": "paris", "relation": "located_in",  "object": "france"},
    ]
})
```

## Querying facts

```python
# All attributes of paris
result = client.perform("data-grout@1/logic.query@1", {
    "namespace": "geography",
    "query": "attribute(paris, Attr, Value)"
})
# -> [{"Attr": "population", "Value": 2161000}, {"Attr": "is_capital", "Value": true}]

# Which cities are capitals of which countries?
result = client.perform("data-grout@1/logic.query@1", {
    "namespace": "geography",
    "query": "relation(City, capital_of, Country)"
})
# -> [{"City": "paris", "Country": "france"}]

# Natural language query: LLM translates to Prolog
result = client.perform("data-grout@1/logic.query@1", {
    "namespace": "geography",
    "query": "What are all the capital cities?",
    "mode": "semantic"
})
```

## Derived facts with rules

```python
# Assert a rule: large capitals
client.perform("data-grout@1/logic.assert@1", {
    "namespace": "geography",
    "facts": [{
        "type": "rule",
        "head": "major_capital(City)",
        "body": "relation(City, capital_of, _), attribute(City, population, Pop), Pop > 1000000"
    }]
})

# Query the derived fact
result = client.perform("data-grout@1/logic.query@1", {
    "namespace": "geography",
    "query": "major_capital(City)"
})
# -> [{"City": "paris"}]
```

## Inline attributes shorthand

Expand an entity's attributes in one fact instead of one fact per attribute:

```python
client.perform("data-grout@1/logic.assert@1", {
    "namespace": "geography",
    "facts": [
        {"type": "entity", "name": "berlin", "attributes": {
            "population": 3645000,
            "is_capital": True,
            "country": "germany"
        }}
    ]
})
# Expands to: entity(berlin) + 3 attribute facts
```

## Field synonyms

Common field names are normalized before storage, use whichever reads naturally:

```python
# These are all equivalent for a relation fact:
{"type": "relation", "subject": "paris", "relation": "capital_of", "object": "france"}
{"type": "relation", "src":     "paris", "predicate": "capital_of", "dst":    "france"}
{"type": "relation", "from":    "paris", "relation":  "capital_of", "to":     "france"}

# Attribute synonyms:
{"type": "attribute", "entity": "paris", "attribute": "population", "value":  2161000}
{"type": "attribute", "entity": "paris", "attr":      "population", "val":    2161000}
{"type": "attribute", "entity": "paris", "property":  "population", "amount": 2161000}
```

## Cross-session persistence

```python
# Session 1
client.perform("data-grout@1/logic.assert@1", {
    "namespace": "my-knowledge-base",
    "facts": [{"type": "entity", "name": "project-alpha", "attributes": {
        "status": "active", "owner": "alice"
    }}]
})

# Session 2: new process, new LLM context — facts are still there
result = client.perform("data-grout@1/logic.query@1", {
    "namespace": "my-knowledge-base",
    "query": "attribute(project-alpha, status, Status)"
})
# -> [{"Status": "active"}]
```

## List namespaces

```python
worlds = client.perform("data-grout@1/logic.worlds@1", {})
for w in worlds["namespaces"]:
    print(f"{w['name']}: {w['fact_count']} facts")
```

## TypeScript

```typescript
import { Client } from '@datagrout/conduit';

const client = new Client({ url: 'https://app.datagrout.ai/servers/<uuid>/mcp' });

await client.perform('data-grout@1/logic.assert@1', {
  namespace: 'geography',
  facts: [
    { type: 'entity',    name: 'paris' },
    { type: 'attribute', entity: 'paris', attribute: 'population', value: 2161000 },
    { type: 'relation',  subject: 'paris', relation: 'capital_of', object: 'france' },
  ],
});

const result = await client.perform('data-grout@1/logic.query@1', {
  namespace: 'geography',
  query: 'relation(City, capital_of, Country)',
});
console.log(result);
```

## Via Claude Code (MCP)

```
Use logic.assert to assert facts about "paris": population 2161000, capital of france.
Then use logic.query with namespace "geography" to find all capitals.
```

## See also

- [batteries/getting-started](../../batteries/getting-started/) — pre-built Prolog rule modules
- [batteries/composing-batteries](../../batteries/composing-batteries/) — combine batteries in one namespace
- [forensic/causal-chain-tracing](../../forensic/causal-chain-tracing/) — reasoning over LC facts
- [first-logic-cell](../../../quickstart/first-logic-cell/) — quickstart version of this recipe
