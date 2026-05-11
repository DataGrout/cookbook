# Recipe: Invariant Code Review

> **Difficulty:** Intermediate  
> **Credits:** 2–5 per review  
> **Time:** ~10 minutes

## What it does

Invariant gives an AI agent a structured, goal-anchored code review workflow. Instead of asking an LLM "is this code good?", you provide:
- A **goal** — what the change is supposed to accomplish
- **Acceptance criteria** — what must be true after the change
- **Constraints** — invariants that must never be broken

The tool returns a per-criterion verdict and a pass/fail gate decision.

Pipeline:
```
code_lens (index)  ->  diff_analyzer (check intent)  ->  review (gate)
                                                              ↓
                                               code_query (audit patterns)
```

## Tools used

| Tool | Purpose | Credits |
|------|---------|---------|
| `invariant.code_lens` | Index source code into semantic facts | ~1–2 |
| `invariant.diff_analyzer` | Check a diff for alignment with stated goal | ~1–2 |
| `invariant.review` | Structured pass/fail verdict against criteria | ~2–3 |
| `invariant.code_query` | Audit a codebase for patterns (cycles, security, etc.) | ~1 |

## The Recipe

### Step 1: Index your code with code_lens

`code_lens` extracts functions, calls, dependencies, and inferred intent. Index once; review queries use the indexed facts.

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

with open("auth.py") as f:
    source = f.read()

lens = client.perform("data-grout@1/invariant.code_lens@1", {
    "code": source,
    "language": "python",
    "filepath": "src/auth.py",
    "repo_id": "my-backend",
    "commit_sha": "a1b2c3d",
    "options": {"include_intent": True}
})

print(lens["functions"])    # extracted function names
print(lens["calls"])        # call graph
print(lens["intent"])       # semantic intent summary
```

### Step 2: Analyze a diff

Before committing or merging, run `diff_analyzer` on the diff to check whether the change does what it claims:

```python
before = open("auth_v1.py").read()
after  = open("auth_v2.py").read()

analysis = client.perform("data-grout@1/invariant.diff_analyzer@1", {
    "before": before,
    "after": after,
    "goal": "Add rate limiting to the login endpoint",
    "language": "python",
    "constraints": [
        "Must not change the response schema for successful logins",
        "Must not remove existing authentication checks"
    ]
})

print(analysis["aligned"])              # True/False
print(analysis["unintended_effects"])   # list of side effects found
print(analysis["deleted_features"])     # anything removed unexpectedly
```

### Step 3: Gate a PR with review

`review` is the full structured verdict. Use `mode: "gate"` for strict pass/fail (e.g. CI):

```python
import subprocess

# Get the diff from git
diff = subprocess.check_output(
    ["git", "diff", "main...HEAD"], text=True
)

verdict = client.perform("data-grout@1/invariant.review@1", {
    "unified_diff": diff,
    "goal": "Refactor database connection pool to use async context managers",
    "criteria": [
        "All database calls use the new async pool",
        "No synchronous DB calls remain in the hot path",
        "Connection cleanup happens in finally blocks"
    ],
    "constraints": [
        "Must not change the public API surface",
        "Must not reduce test coverage"
    ],
    "language": "python",
    "repo_id": "my-backend",
    "mode": "gate"     # "check" for lenient, "gate" for strict
})

print(verdict["pass"])          # True / False
print(verdict["verdict"])       # "pass" | "fail"
for c in verdict["criteria_results"]:
    status = "✓" if c["pass"] else "✗"
    print(f"  {status} {c['criterion']}: {c['note']}")
```

Output:
```
  ✓ All database calls use the new async pool
  ✓ No synchronous DB calls remain in the hot path
  ✗ Connection cleanup happens in finally blocks: Found 2 locations without finally
```

### Step 4: Audit with code_query

After indexing multiple files, use `code_query` to ask structural questions:

```python
# Find dependency cycles
cycles = client.perform("data-grout@1/invariant.code_query@1", {
    "repo_id": "my-backend",
    "query": "dependency_cycles"
})
for cycle in cycles["cycles"]:
    print(f"Cycle: {' -> '.join(cycle)}")

# Find functions with no tests
gaps = client.perform("data-grout@1/invariant.code_query@1", {
    "repo_id": "my-backend",
    "query": "test_gaps"
})
print(f"Untested: {gaps['untested_functions']}")

# Security concerns
sec = client.perform("data-grout@1/invariant.code_query@1", {
    "repo_id": "my-backend",
    "query": "security_concerns"
})
print(sec["findings"])
```

Available query types:
- `dependency_cycles` — circular imports/dependencies
- `intent_mismatches` — code intent disagrees with function name
- `test_gaps` — functions with no corresponding tests
- `high_risk_changes` — functions changed frequently (hotspots)
- `debug_in_prod` — logging/debug calls in production paths
- `security_concerns` — hardcoded secrets, SQL construction, unsafe evals
- `hotspots` — high-churn, high-complexity functions
- `orphans` — functions never called

## Full CI gate pipeline

```python
import subprocess, sys
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

# 1. Get the diff
diff = subprocess.check_output(["git", "diff", "main...HEAD"], text=True)
if not diff.strip():
    print("No changes."); sys.exit(0)

# 2. Index changed files
changed_files = subprocess.check_output(
    ["git", "diff", "--name-only", "main...HEAD"], text=True
).splitlines()

for filepath in changed_files:
    if filepath.endswith(".py"):
        try:
            source = open(filepath).read()
            client.perform("data-grout@1/invariant.code_lens@1", {
                "code": source, "language": "python",
                "filepath": filepath, "repo_id": "my-backend"
            })
        except FileNotFoundError:
            pass  # deleted file

# 3. Review gate
verdict = client.perform("data-grout@1/invariant.review@1", {
    "unified_diff": diff,
    "goal": "Add JWT refresh token support",
    "criteria": [
        "New /auth/refresh endpoint exists",
        "Refresh tokens are stored in httpOnly cookies",
        "Old access tokens are invalidated on refresh"
    ],
    "constraints": [
        "Must not break existing /auth/login flow",
        "Must not log token values"
    ],
    "repo_id": "my-backend",
    "mode": "gate"
})

if not verdict["pass"]:
    print("❌ Review failed:")
    for c in verdict["criteria_results"]:
        if not c["pass"]:
            print(f"  ✗ {c['criterion']}: {c['note']}")
    sys.exit(1)

print("✓ Review passed")
sys.exit(0)
```

## Using with diff from string (no git)

```python
before = """
def login(username, password):
    user = db.get_user(username)
    if user and user.password == password:
        return generate_token(user.id)
    return None
"""

after = """
def login(username, password):
    user = db.get_user(username)
    if user and bcrypt.checkpw(password.encode(), user.password_hash):
        return generate_token(user.id)
    return None
"""

verdict = client.perform("data-grout@1/invariant.review@1", {
    "before": before,
    "after": after,
    "goal": "Replace plaintext password comparison with bcrypt",
    "criteria": [
        "bcrypt.checkpw is used for password verification",
        "No plaintext password comparison remains"
    ],
    "language": "python",
    "mode": "check"
})
```

## See also

- [code-quality-gate](../../combined/code-quality-gate/) — full pipeline combining invariant + warden
- [warden/prompt-injection-defense](../../warden/prompt-injection-defense/) — safety checks for untrusted content
- [forensic/causal-chain-tracing](../../forensic/causal-chain-tracing/) — tracing causation after a bug
