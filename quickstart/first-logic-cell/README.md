# Recipe: First Logic Cell

## What it does

Assert facts into a Logic Cell, query them with Prolog, then reflect on the namespace. Introduction to symbolic memory, facts that persist across sessions and are queryable without LLM inference.

## Tools used

- `logic.assert` — write facts to a namespace
- `logic.query` — Prolog query over stored facts
- `logic.worlds` — list active namespaces

## Credits cost

1–2 credits (assert: 1, query: 0 if cached, worlds: 0).

## The Recipe

### Asserting facts

Facts have a `type` field that determines their structure:

| type | required fields | use for |
|------|----------------|---------|
| `entity` | `name` | named things |
| `attribute` | `entity`, `attribute`, `value` | properties of entities |
| `relation` | `subject`, `relation`, `object` | relationships between entities |
| `context` | `key`, `value` | session/task context |
| `rule` | `head`, `body` | derived facts (Prolog clauses) |

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

# Assert facts into a namespace
client.logic.assert_facts(
    facts=[
        {"type": "entity", "name": "paris"},
        {"type": "entity", "name": "france"},
        {"type": "attribute", "entity": "paris", "attribute": "population", "value": 2_161_000},
        {"type": "attribute", "entity": "paris", "attribute": "is_capital", "value": True},
        {"type": "relation", "subject": "paris", "relation": "capital_of", "object": "france"},
        {"type": "relation", "subject": "paris", "relation": "located_in", "object": "france"},
    ],
    namespace="geography"
)
```

### Querying facts

```python
# Simple query
result = client.logic.query(
    query="attribute(paris, Attr, Value)",
    namespace="geography"
)
# Returns: [{"Attr": "population", "Value": 2161000}, {"Attr": "is_capital", "Value": true}]

# Relation query
result = client.logic.query(
    query="relation(City, capital_of, Country)",
    namespace="geography"
)
# Returns: [{"City": "paris", "Country": "france"}]

# Natural language query (uses NL->Prolog translation)
result = client.logic.query(
    query="What are all the cities that are capitals?",
    namespace="geography",
    nl=True
)
```

### Rules: derived facts

```python
# Assert a rule
client.logic.assert_facts(
    facts=[{
        "type": "rule",
        "head": "important_city(City)",
        "body": "attribute(City, is_capital, true), attribute(City, population, Pop), Pop > 1000000"
    }],
    namespace="geography"
)

# Query the derived fact
result = client.logic.query(
    query="important_city(City)",
    namespace="geography"
)
# Returns: [{"City": "paris"}]
```

### Listing namespaces

```python
worlds = client.logic.worlds()
for w in worlds["namespaces"]:
    print(f"{w['name']}: {w['fact_count']} facts, last updated {w['updated_at']}")
```

## Cross-session persistence

Facts asserted in one session are available in all future sessions:

```python
# Session 1: assert
client.logic.assert_facts(facts=[...], namespace="my-knowledge-base")

# Session 2 (new process, new context window): query
result = client.logic.query(query="...", namespace="my-knowledge-base")
# Facts are still there
```

## TypeScript

```typescript
import { Client } from '@datagrout/conduit';

const client = new Client({ apiKey: 'your-key' });

// Assert facts
await client.perform('data-grout@1/logic.assert@1', {
    facts: [
        { type: 'entity', name: 'paris' },
        { type: 'entity', name: 'france' },
        { type: 'attribute', entity: 'paris', attribute: 'population', value: 2161000 },
        { type: 'attribute', entity: 'paris', attribute: 'is_capital', value: true },
        { type: 'relation', subject: 'paris', relation: 'capital_of', object: 'france' }
    ],
    namespace: 'geography'
});

// Query
const result = await client.perform('data-grout@1/logic.query@1', {
    query: 'attribute(paris, Attr, Value)',
    namespace: 'geography'
});
console.log(result.bindings);
// [{ Attr: 'population', Value: 2161000 }, { Attr: 'is_capital', Value: true }]

// Assert a rule
await client.perform('data-grout@1/logic.assert@1', {
    facts: [{
        type: 'rule',
        head: 'important_city(City)',
        body: 'attribute(City, is_capital, true), attribute(City, population, Pop), Pop > 1000000'
    }],
    namespace: 'geography'
});

// Query the derived fact
const derived = await client.perform('data-grout@1/logic.query@1', {
    query: 'important_city(City)',
    namespace: 'geography'
});
console.log(derived.bindings); // [{ City: 'paris' }]

// List namespaces
const worlds = await client.perform('data-grout@1/logic.worlds@1', {});
worlds.namespaces.forEach((w: any) => {
    console.log(`${w.name}: ${w.fact_count} facts, last updated ${w.updated_at}`);
});
```

## Try it

```python
python first_logic_cell.py
# Then start a new Python process and query the same namespace
python query_session.py
```

## Variations

**Inline attributes**, entity with attributes map expands automatically:
```python
{"type": "entity", "name": "paris", "attributes": {"population": 2161000, "is_capital": True}}
# Expands to: entity fact + 2 attribute facts
```

**Field synonyms**, common aliases are normalized:
```python
{"type": "relation", "src": "paris", "predicate": "capital_of", "dst": "france"}
# Same as: subject, relation, object
```

## Next steps

- [assert-and-query](../../recipes/logic/assert-and-query/) — full fact type reference and query patterns
- [forensic/causal-chain-tracing](../../recipes/forensic/causal-chain-tracing/) — reasoning over facts
