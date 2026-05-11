# recipes

Recipes and patterns for building with [DataGrout](https://datagrout.ai). Each recipe is a self-contained example showing how to use DG tools to solve a specific class of problem.

Organized by complexity: **quickstart -> patterns -> recipes -> agents -> advanced**.

## Structure

```
recipes/
├── quickstart/             # Entry-level examples to get started
│   ├── hello-world/        # Connect and call one tool
│   ├── first-flow/         # Compose two tools with flow.into
│   ├── first-logic-cell/   # Assert, query, reflect
│   ├── understanding-cache-refs/ # Free data piping
│   └── discovery/          # Discover tools by goal, plan workflows
├── patterns/               # Reusable cross-cutting patterns (transport, auth, pipeline shapes)
├── recipes/                # Suite-specific tool recipes
│   ├── logic/              # Symbolic memory: assert, query, constrain, reflect
│   ├── batteries/          # Pre-built Prolog rule modules (install, validate, compose)
│   ├── flow/               # Workflow orchestration (flow.into, flow.route)
│   ├── data-frame-math/    # Zero-credit data analysis: stats, trend, outliers, correlation
│   ├── prism/              # Semantic reshape and visualization
│   ├── inference/          # Grounded web research
│   ├── warden/             # Safety and injection detection (canary, intent, adjudicate)
│   ├── invariant/          # Code analysis, review gates, codebase audits
│   ├── latent/             # Concept exploration
│   ├── forensic/           # Causal investigation
│   └── combined/           # Multi-suite pipelines
├── agents/                 # Full agent FSM examples (Agentsmith SDK)
├── advanced/               # Skills, custom tools, multi-server demux
└── games/                  # Prebuilt Prolog rulesets for Roblox / Tether
    └── rulesets/           # installable modules (inventory, quests, loot-tables, …)
```

## Prerequisites

Sign up at **[app.datagrout.ai](https://app.datagrout.ai)** to get your server URL and API key.

```bash
# Install the Conduit SDK for your language
pip install datagrout-conduit          # Python
npm install @datagrout/conduit         # TypeScript
cargo add datagrout-conduit            # Rust
mix deps.get datagrout_conduit         # Elixir
gem install datagrout-conduit          # Ruby

# Or use Claude Code with the DataGrout MCP server
# Get your MCP URL from app.datagrout.ai -> Settings -> MCP
claude mcp add datagrout https://app.datagrout.ai/servers/<uuid>/mcp
```

## Recipe Template

See [_template.md](./_template.md) for the standard recipe format.

## Quick Index

### Quickstart
- [hello-world](./quickstart/hello-world/) — Connect, discover, call one tool
- [first-flow](./quickstart/first-flow/) — Compose two tools with flow.into
- [first-logic-cell](./quickstart/first-logic-cell/) — Assert, query, reflect
- [understanding-cache-refs](./quickstart/understanding-cache-refs/) — How cache_ref flows through a pipeline and why data transfer is free
- [discovery](./quickstart/discovery/) — Discover tools by goal, plan multi-step workflows, agent self-orientation

### Patterns
- [zero-credit-pipeline](./patterns/zero-credit-pipeline/) — Data transforms at zero tokens
- [cache-ref-composition](./patterns/cache-ref-composition/) — Pipe results via cache_ref
- [background-tasks](./patterns/background-tasks/) — Non-blocking long-running calls
- [onramp-bootstrap](./patterns/onramp-bootstrap/) — Autonomous agent self-registration and mTLS bootstrap
- [jsonrpc-transport](./patterns/jsonrpc-transport/) — JSONRPC and WebSocket transport vs MCP
- [scheduled-monitoring](./patterns/scheduled-monitoring/) — Periodic triggers with Governor
- [refract-reshape](./patterns/refract-reshape/) — Semantic data transformation

### Inference Recipes
- [grounded-search](./recipes/inference/grounded-search/) — inference.search, .research, .rfi — quality levels, citations, background promotion

### Prism Recipes
- [analyze-modes](./recipes/prism/analyze-modes/) — exploratory, competitive, causal, and deep analysis modes

### Latent Recipes
- [concept-exploration](./recipes/latent/concept-exploration/) — latent.orient and latent.horizon for semantic positioning and blind spot discovery

### Logic Cell Recipes
- [assert-and-query](./recipes/logic/assert-and-query/) — Assert entity/attribute/relation facts, query with Prolog
- [logic/constrain](./recipes/logic/constrain/) — Custom Prolog rules: derived facts, transitive graphs, business logic

### Batteries Recipes
- [batteries/getting-started](./recipes/batteries/getting-started/) — Search, describe, install, and query a battery
- [batteries/composing-batteries](./recipes/batteries/composing-batteries/) — Combine inventory + quests + loot-tables in one namespace
- [batteries/validate](./recipes/batteries/validate/) — Diagnose installation: per-predicate pass/fail, missing fact hints, optional CTC

### Flow Recipes
- [flow/conditional-routing](./recipes/flow/conditional-routing/) — flow.route patterns: predicates, AND branches, catch-all, cache_ref
- [flow/log-processing](./recipes/flow/log-processing/) — flow.into + conditional step for log severity routing
- [flow/trend-analysis-pipeline](./recipes/flow/trend-analysis-pipeline/) — filter -> group -> trend -> analyze at minimal cost

### Data-Frame / Math Recipes
- [data-frame-math/csv-analysis](./recipes/data-frame-math/csv-analysis/) — Descriptive stats, trend, outlier detection, correlation — all zero credits

### Warden Recipes
- [warden/prompt-injection-defense](./recipes/warden/prompt-injection-defense/) — Three-tier injection detection: canary -> intent -> adjudicate

### Invariant Recipes
- [invariant/code-review](./recipes/invariant/code-review/) — Goal-anchored code review, CI gate, codebase audits (cycles, security, test gaps)

### Docs Recipes
- [docs](./recipes/docs/) — Create, update, retrieve, and delete persistent documents; agent working memory; cleanup with approval gate

### Notable Combined Recipes
- [competitive-intel-pipeline](./recipes/combined/competitive-intel-pipeline/) — search -> refract -> assert -> tabulate -> analyze
- [research-to-report](./recipes/combined/research-to-report/) — Full research -> structured report pipeline
- [code-quality-gate](./recipes/combined/code-quality-gate/) — lens -> review -> warden -> route

### Games / Roblox Rulesets ([Tether](https://github.com/datagrout/tether))
- [games/](./games/) — LC mental model, module catalog, and game loop patterns
- [inventory](./games/rulesets/inventory/) — Item carrying, weight limits, slot constraints
- [loot-tables](./games/rulesets/loot-tables/) — Rarity tiers, conditions, drop chance calculation
- [quests](./games/rulesets/quests/) — Quest availability, prerequisites, objective tracking

[Tether](https://github.com/datagrout/tether) is the Roblox/Luau client library for DataGrout. Install any battery from Luau with one line:
```lua
dg:batteries().install("quests", "my-game", function(r) print(r.predicate_count .. " predicates installed") end)
```

### Lumen

[Lumen](https://github.com/DataGrout/lumen) is a free token usage monitor and cost tracker. On macOS it runs as a native status bar app; on Linux and Windows the same `lumen-core` Rust daemon serves a browser dashboard at `http://127.0.0.1:9091/dashboard`. When you have Lumen running alongside a DG session, the `lumen` tool suite lets agents read live session data:

- `lumen.laps` — retrieve lap history and per-lap token/cost breakdown
- `lumen.compare` — compare two laps or time windows side-by-side
- `lumen.dashboard` — current session totals and rolling metrics

These tools read from the `_lumen_<subscriber_id>` LC namespace and cost zero credits. Useful for validating that pipeline steps run at zero tokens (see [zero-credit-pipeline](./patterns/zero-credit-pipeline/)).

### Benchmark Cross-References
Recipes link to [benchmarks](https://github.com/DataGrout/benchmarks) for cost and token benchmarks:
- `forensic/causal-chain-tracing` -> benchmark 01 (multi-hop debugging)
- `combined/competitive-intel-pipeline` -> benchmark 05
- `data-frame-math/csv-analysis` -> benchmark 04
- `invariant/code-review` -> benchmark 03
