# Advanced: Forensic Rules Library

## What it does

The complete set of forensic Prolog rules: `causal_chain`, `blast_radius`, `all_affected`, `timeout_risk`, `invariant_holds`, `invariant_violated`, `contradiction`. Ready to assert into any `_forensic` namespace.

## The Rules

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

FORENSIC_RULES = [
    # --- Causation ---

    # Direct cause
    {"type": "rule", "head": "causes(A, B)", "body": "observed(A), leads_to(A, B)"},

    # Transitive causal chain
    {"type": "rule", "head": "causal_chain(Root, Symptom)", "body": "causes(Root, Symptom)"},
    {"type": "rule",
     "head": "causal_chain(Root, Symptom)",
     "body": "causes(Root, Mid), causal_chain(Mid, Symptom)"},

    # --- Blast radius ---

    # All nodes reachable from a root cause
    {"type": "rule",
     "head": "all_affected(Root, Effects)",
     "body": "findall(E, causal_chain(Root, E), Effects)"},

    # Count of affected nodes
    {"type": "rule",
     "head": "blast_radius(Root, N)",
     "body": "all_affected(Root, Effects), length(Effects, N)"},

    # --- Root cause identification ---

    # A root cause is observed but has nothing causing it
    {"type": "rule",
     "head": "root_cause(X)",
     "body": "observed(X), \\+ causes(_, X)"},

    # A leaf symptom causes nothing further
    {"type": "rule",
     "head": "leaf_symptom(X)",
     "body": "causes(_, X), \\+ causes(X, _)"},

    # --- Timeout risk ---

    # Risk: app processing time exceeds proxy idle timeout
    {"type": "rule",
     "head": "timeout_risk(Handler)",
     "body": ("config(reverse_proxy, idle_timeout_ms, ProxyTimeout), "
              "max_processing_time(Handler, AppTimeout), "
              "AppTimeout > ProxyTimeout")},

    # Risk with margin (fail if within 20% of limit)
    {"type": "rule",
     "head": "timeout_risk_marginal(Handler)",
     "body": ("config(reverse_proxy, idle_timeout_ms, ProxyTimeout), "
              "max_processing_time(Handler, AppTimeout), "
              "Margin is ProxyTimeout * 0.8, "
              "AppTimeout > Margin")},

    # --- Invariant checking ---

    # An invariant holds if its condition is true
    {"type": "rule",
     "head": "invariant_holds(Name)",
     "body": "invariant(Name, Condition), call(Condition)"},

    # An invariant is violated if it is defined but does not hold
    {"type": "rule",
     "head": "invariant_violated(Name)",
     "body": "invariant(Name, _), \\+ invariant_holds(Name)"},

    # List all violated invariants
    {"type": "rule",
     "head": "all_violations(Violations)",
     "body": "findall(N, invariant_violated(N), Violations)"},

    # --- Contradiction detection ---

    # Two facts are contradictory if both are asserted and they conflict
    {"type": "rule",
     "head": "contradiction(Fact1, Fact2)",
     "body": "attribute(E, Attr, V1), attribute(E, Attr, V2), V1 \\= V2, Fact1 = attr(E, Attr, V1), Fact2 = attr(E, Attr, V2)"},

    # --- Health check coverage ---

    # A slow path is not covered if health check skips it
    {"type": "rule",
     "head": "uncovered_slow_path(Handler)",
     "body": "slow_path(Handler), \\+ health_check_covers(Handler)"},
]


def assert_forensic_rules(client, system_name: str) -> str:
    """Assert all forensic rules into a system-specific namespace."""
    namespace = f"_forensic_{system_name}"
    client.logic.assert_facts(facts=FORENSIC_RULES, namespace=namespace)
    print(f"Asserted {len(FORENSIC_RULES)} forensic rules into {namespace}")
    return namespace
```

## Usage

```python
# 1. Assert the rules library
namespace = assert_forensic_rules(client, "myapp")

# 2. Assert your system-specific facts
client.logic.assert_facts(facts=[
    {"type": "relation", "subject": "buffered_mode", "relation": "leads_to", "object": "no_keepalives"},
    {"type": "relation", "subject": "no_keepalives", "relation": "leads_to", "object": "proxy_timeout"},

    # System invariants
    {"type": "rule",
     "head": "invariant(proxy_protection, sends_keepalives(Mode, true))",
     "body": "delivery_mode(Handler, Mode)"},
], namespace=namespace)

# 3. Query
violations = client.logic.query(query="all_violations(Vs)", namespace=namespace)
chain = client.logic.query(query="causal_chain(buffered_mode, X)", namespace=namespace)
risk = client.logic.query(query="timeout_risk(Handler)", namespace=namespace)
```

## Pre-built namespace

The rules are also available as a pre-built namespace you can clone:

```python
# Clone the library into your namespace
client.logic.clone(
    source="_forensic_template",
    target=f"_forensic_{your_system}"
)
```

## See also

- [causal-chain-tracing](../../recipes/forensic/causal-chain-tracing/) — basic usage
- [benchmarks/01-multi-hop-debugging](https://github.com/DataGrout/benchmarks) — the benchmark this supports
