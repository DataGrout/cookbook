# Recipe: Causal Chain Tracing

## What it does

Assert causal facts about a system, then query `causal_chain/2` to trace from symptom to root cause. The core Forensic Investigation (FI) pattern.

## Tools used

- `logic.assert` — write causal facts to a forensic namespace
- `logic.query` — run Prolog over asserted facts

## Credits cost

1 credit (assert). Query: 0 if cached.

## Background: Forensic Prolog Rules

FI uses a standard library of Prolog rules that you assert once into a `_forensic` namespace. The key rules:

```prolog
% Direct causation
causes(A, B) :- observed(A), leads_to(A, B).

% Transitive chain
causal_chain(Root, Symptom) :-
    causes(Root, Symptom).
causal_chain(Root, Symptom) :-
    causes(Root, Intermediate),
    causal_chain(Intermediate, Symptom).

% Blast radius (all downstream effects)
all_affected(Root, Effects) :-
    findall(E, causal_chain(Root, E), Effects).
```

## The Recipe

### Step 1: Assert the forensic rules

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

forensic_rules = [
    # Direct causation
    {
        "type": "rule",
        "head": "causes(A, B)",
        "body": "observed(A), leads_to(A, B)"
    },
    # Transitive chain
    {
        "type": "rule",
        "head": "causal_chain(Root, Symptom)",
        "body": "causes(Root, Symptom)"
    },
    {
        "type": "rule",
        "head": "causal_chain(Root, Symptom)",
        "body": "causes(Root, Intermediate), causal_chain(Intermediate, Symptom)"
    },
    # Blast radius
    {
        "type": "rule",
        "head": "all_affected(Root, Effects)",
        "body": "findall(E, causal_chain(Root, E), Effects)"
    }
]

client.logic.assert_facts(facts=forensic_rules, namespace="_forensic_myapp")
```

### Step 2: Assert observed system facts

```python
# What you observed in the logs / code
observations = [
    # Observed symptoms
    {"type": "entity", "name": "observed", "attributes": {
        "buffered_response_handler": True,
        "missing_account_id_index": True
    }},

    # Causal links
    {"type": "relation", "subject": "buffered_response_handler",
     "relation": "leads_to", "object": "no_bytes_sent_during_processing"},
    {"type": "relation", "subject": "no_bytes_sent_during_processing",
     "relation": "leads_to", "object": "nginx_idle_timeout_fires"},
    {"type": "relation", "subject": "missing_account_id_index",
     "relation": "leads_to", "object": "slow_query_32s"},
    {"type": "relation", "subject": "slow_query_32s",
     "relation": "leads_to", "object": "no_bytes_sent_during_processing"},
    {"type": "relation", "subject": "nginx_idle_timeout_fires",
     "relation": "leads_to", "object": "client_receives_504"}
]

client.logic.assert_facts(facts=observations, namespace="_forensic_myapp")
```

### Step 3: Trace the chain

```python
# Trace from root cause to symptom
result = client.logic.query(
    query="causal_chain(missing_account_id_index, client_receives_504)",
    namespace="_forensic_myapp"
)
print(result)  # True, confirms the chain

# Find all steps in the chain
result = client.logic.query(
    query="causal_chain(missing_account_id_index, Effect)",
    namespace="_forensic_myapp"
)
for r in result:
    print(r["Effect"])
# slow_query_32s
# no_bytes_sent_during_processing
# nginx_idle_timeout_fires
# client_receives_504

# Find all things affected by a given root cause
result = client.logic.query(
    query="all_affected(missing_account_id_index, Effects)",
    namespace="_forensic_myapp"
)
print(result[0]["Effects"])
```

### Step 4: Identify root causes

```python
# What is NOT a consequence of something else? That's a root cause.
result = client.logic.query(
    query="leads_to(_, X), \\+ leads_to(_, X)",  # has no incoming cause
    namespace="_forensic_myapp"
)
# This query finds all nodes with no incoming edges: the root causes
```

## Complete working example

The benchmark 01 broken project is a full worked example:

```python
# See benchmarks/01-multi-hop-debugging/with-dg/
# The full FI investigation using this pattern
```

## Variations

**Invariant violation detection:**
```python
# Assert invariant rules
client.logic.assert_facts(facts=[{
    "type": "rule",
    "head": "invariant_violated(proxy_protection)",
    "body": "delivery_mode(_, buffered), \\+ sends_keepalives(buffered, true)"
}], namespace="_forensic_myapp")

result = client.logic.query(
    query="invariant_violated(Invariant)",
    namespace="_forensic_myapp"
)
```

**Timeout risk detection:**
```python
client.logic.assert_facts(facts=[{
    "type": "rule",
    "head": "timeout_risk(Handler)",
    "body": "config(reverse_proxy, idle_timeout_ms, ProxyTimeout), "
            "max_processing_time(Handler, AppTimeout), "
            "AppTimeout > ProxyTimeout"
}], namespace="_forensic_myapp")
```

## See also

- [forensic-rules-library](../../advanced/forensic-rules-library/) — complete rule set, ready to assert
- [benchmarks/01-multi-hop-debugging](https://github.com/DataGrout/benchmarks) — benchmark using this pattern
