# Recipe: Code Quality Gate

> **Difficulty:** Intermediate  
> **Credits:** 5–15  
> **Time:** ~10 minutes

## What it does

Run a submitted code change through a multi-stage quality gate: static analysis, LLM code review, safety scan, then route to approve/request-changes/block based on the combined verdict. Each stage adds signal; `flow.route` makes the final dispatch decision.

Pipeline: `invariant.lens` -> `invariant.review` -> `warden.scan` -> `flow.route`

## Tools used

| Tool | Credits | Purpose |
|------|---------|---------|
| `invariant.lens` | 1–3 | Static analysis — complexity, patterns, issues |
| `invariant.review` | 2–5 | LLM code review against standards |
| `warden.scan` | 1 | Safety scan — injections, secrets, dangerous patterns |
| `flow.route` | 0 | Dispatch based on combined verdict |

## The Recipe

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

# The diff or file contents to review
code = """
def process_user_query(query: str, db_conn) -> dict:
    sql = f"SELECT * FROM users WHERE name = '{query}'"
    result = db_conn.execute(sql)
    return {"users": result.fetchall()}
"""

# Step 1: Static analysis
lens = client.perform("data-grout@1/invariant.lens@1", {
    "code": code,
    "checks": ["complexity", "patterns", "anti_patterns", "dependencies"]
})

# Step 2: LLM review
review = client.perform("data-grout@1/invariant.review@1", {
    "code": code,
    "lens_result": lens["cache_ref"],
    "standards": [
        "No raw string interpolation in SQL queries",
        "Input validation required at all API boundaries",
        "Secrets must not appear in code or logs"
    ],
    "focus": "security correctness"
})

# Step 3: Safety scan
scan = client.perform("data-grout@1/warden.scan@1", {
    "content": code,
    "checks": ["injection", "secrets", "dangerous_patterns", "data_exfiltration"]
})

# Step 4: Route based on combined verdict
verdict = {
    "has_critical": scan["has_violations"] or review["verdict"] == "block",
    "has_warnings": lens["issue_count"] > 0 or review["verdict"] == "request_changes",
    "complexity_score": lens["complexity_score"],
    "review_verdict": review["verdict"],
    "scan_clean": not scan["has_violations"]
}

result = client.perform("data-grout@1/flow.route@1", {
    "payload": verdict,
    "branches": [
        {
            "when": [{"field": "has_critical", "op": "eq", "value": True}],
            "label": "block",
            "then": {
                "tool": "data-grout@1/prism.analyze@1",
                "args": {
                    "goal": "Write a clear blocking comment explaining the critical issue(s) "
                            "and the exact changes needed to unblock.",
                    "data": "$payload",
                    "mode": "deductive"
                }
            }
        },
        {
            "when": [{"field": "has_warnings", "op": "eq", "value": True}],
            "label": "request_changes",
            "then": {
                "tool": "data-grout@1/prism.analyze@1",
                "args": {
                    "goal": "Write a constructive review comment listing issues by priority. "
                            "Be specific about what to change and why.",
                    "data": "$payload",
                    "mode": "deductive"
                }
            }
        },
        {
            "when": "_",
            "label": "approve",
            "then": {
                "tool": "data-grout@1/prism.refract@1",
                "args": {
                    "goal": "Extract: verdict='approved', summary of what looks good",
                    "data": "$payload"
                }
            }
        }
    ]
})

print(f"Verdict: {result['label']}")
print(result['result'])
```

## As a flow.into pipeline

```python
result = client.perform("data-grout@1/flow.into@1", {
    "plan": [
        {
            "tool": "data-grout@1/invariant.lens@1",
            "output": "lens",
            "args": {
                "code": code_diff,
                "checks": ["complexity", "patterns", "anti_patterns"]
            }
        },
        {
            "tool": "data-grout@1/invariant.review@1",
            "output": "review",
            "args": {
                "code": code_diff,
                "lens_result": "$lens",
                "focus": "security correctness maintainability"
            }
        },
        {
            "tool": "data-grout@1/warden.scan@1",
            "output": "scan",
            "args": {
                "content": code_diff,
                "checks": ["injection", "secrets", "dangerous_patterns"]
            }
        },
        {
            "type": "conditional",
            "input": "scan",
            "output": "gate_result",
            "branches": [
                {
                    "when": [{"field": "has_violations", "op": "eq", "value": True}],
                    "label": "block",
                    "then": {
                        "tool": "data-grout@1/prism.analyze@1",
                        "args": {
                            "goal": "Write a blocking PR comment explaining the security issue(s) "
                                    "and required remediation steps.",
                            "data": "$scan",
                            "mode": "deductive"
                        }
                    }
                }
            ],
            "else": {
                "tool": "data-grout@1/prism.analyze@1",
                "args": {
                    "goal": "Combine the lens analysis and code review into a final PR review comment. "
                            "List issues by severity. Recommend approve or request-changes.",
                    "data": "$review",
                    "mode": "deductive"
                }
            }
        }
    ]
})

print(result["execution_result"]["gate_result"])
```

## Varying the standards

Pass project-specific standards to `invariant.review` to enforce team conventions:

```python
review = client.perform("data-grout@1/invariant.review@1", {
    "code": code_diff,
    "standards": [
        "All database queries must use parameterized statements",
        "Functions longer than 40 lines require a docstring",
        "No direct os.environ access — use config.get() from our settings module",
        "Error messages must not include stack traces in production code paths",
    ],
    "focus": "correctness security conventions"
})
```

## Warden violation types

`warden.scan` checks include:

| Check | Catches |
|-------|---------|
| `injection` | SQL injection, command injection, path traversal |
| `secrets` | Hardcoded API keys, passwords, tokens in code |
| `dangerous_patterns` | `eval`, `exec`, `pickle.loads`, unsafe deserializers |
| `data_exfiltration` | Logging of PII, passwords, tokens |
| `prompt_injection` | User input passed unsanitized to LLM prompts |

```python
# Check only for secrets and injections (faster)
scan = client.perform("data-grout@1/warden.scan@1", {
    "content": code,
    "checks": ["injection", "secrets"]
})
```

## CI integration pattern

```python
import subprocess
import sys

# Get the diff from git
diff = subprocess.check_output(["git", "diff", "main...HEAD"]).decode()

result = client.perform("data-grout@1/flow.into@1", {
    "plan": [lens_step, review_step, scan_step, gate_step]
})

verdict = result["execution_result"]["gate_result"]

if verdict["label"] == "block":
    print("BLOCKED:", verdict["result"])
    sys.exit(1)
elif verdict["label"] == "request_changes":
    print("REVIEW REQUIRED:", verdict["result"])
    sys.exit(1)
else:
    print("APPROVED:", verdict["result"])
    sys.exit(0)
```

## See also

- [conditional-routing](../../flow/conditional-routing/) — flow.route patterns in depth
- [forensic/causal-chain-tracing](../../forensic/causal-chain-tracing/) — investigating issues found by the gate
- [advanced/forensic-rules-library](../../../advanced/forensic-rules-library/) — Prolog rules for code invariants
