# Agent Recipe: Research Agent

## What it does

A finite state machine (FSM) agent that runs autonomously: discovers topics, researches them, stores findings, analyzes patterns, and produces a structured report. Uses Agentsmith SDK.

## FSM States

```
idle -> researching -> storing -> analyzing -> reporting -> idle
          ↓ (error)                                     ↑
        failed ─────────────────────────────────────────┘ (retry)
```

## Tools used

- `inference.search` — grounded research
- `prism.refract` — normalize findings to facts
- `logic.assert` — persist findings
- `logic.query` — query accumulated knowledge
- `prism.analyze` — synthesize report
- `docs.create` — save report

## The Recipe

```python
from agentsmith import Agent, State, Transition
from datagrout_conduit import Client

client = Client(api_key="your-key")

class ResearchAgent(Agent):
    initial_state = "idle"

    states = {
        "idle": State(
            on_enter=lambda ctx: print(f"Agent ready. Topics: {ctx.get('topics', [])}"),
            transitions=[
                Transition(to="researching", condition=lambda ctx: bool(ctx.get("topics")))
            ]
        ),

        "researching": State(
            on_enter=lambda ctx: research_next_topic(ctx),
            transitions=[
                Transition(to="storing", condition=lambda ctx: ctx.get("research_done")),
                Transition(to="failed", condition=lambda ctx: ctx.get("research_error"))
            ]
        ),

        "storing": State(
            on_enter=lambda ctx: store_findings(ctx),
            transitions=[
                Transition(to="researching", condition=lambda ctx: ctx.get("more_topics")),
                Transition(to="analyzing", condition=lambda ctx: not ctx.get("more_topics"))
            ]
        ),

        "analyzing": State(
            on_enter=lambda ctx: analyze_findings(ctx),
            transitions=[
                Transition(to="reporting", condition=lambda ctx: ctx.get("analysis_done"))
            ]
        ),

        "reporting": State(
            on_enter=lambda ctx: produce_report(ctx),
            transitions=[
                Transition(to="idle", condition=lambda ctx: ctx.get("report_saved"))
            ]
        ),

        "failed": State(
            on_enter=lambda ctx: handle_failure(ctx),
            transitions=[
                Transition(to="researching", condition=lambda ctx: ctx.get("retry_ok")),
                Transition(to="idle", condition=lambda ctx: not ctx.get("retry_ok"))
            ]
        )
    }


def research_next_topic(ctx: dict) -> None:
    topic = ctx["topics"].pop(0)
    ctx["current_topic"] = topic
    try:
        result = client.inference.search(
            query=f"{topic} {ctx.get('domain', '')}",
            quality="high"
        )
        ctx["current_result"] = result["result"]
        ctx["research_done"] = True
        ctx["research_error"] = False
    except Exception as e:
        ctx["research_error"] = str(e)
        ctx["retry_count"] = ctx.get("retry_count", 0) + 1


def store_findings(ctx: dict) -> None:
    facts = client.prism.refract(
        goal=f"Extract entity and attribute facts about: {ctx['current_topic']}",
        data=ctx["current_result"]
    )
    client.logic.assert_facts(
        facts=facts["result"],
        namespace=ctx["namespace"]
    )
    ctx["stored_topics"] = ctx.get("stored_topics", []) + [ctx["current_topic"]]
    ctx["research_done"] = False
    ctx["more_topics"] = bool(ctx["topics"])


def analyze_findings(ctx: dict) -> None:
    table = client.logic.tabulate(
        query="attribute(Entity, Attr, Value)",
        namespace=ctx["namespace"]
    )
    ctx["analysis"] = client.prism.analyze(
        goal=ctx.get("analysis_goal", "Synthesize findings and identify key patterns"),
        data=table["cache_ref"],
        mode="exploratory"
    )["result"]
    ctx["analysis_done"] = True


def produce_report(ctx: dict) -> None:
    doc = client.docs.create(
        title=f"Research Report: {ctx.get('report_title', 'Findings')}",
        body=ctx["analysis"],
        tags=["research", "agent-generated"]
    )
    ctx["report_ref"] = doc["ref"]
    ctx["report_saved"] = True
    print(f"Report saved: {doc['ref']}")


def handle_failure(ctx: dict) -> None:
    if ctx.get("retry_count", 0) < 3:
        ctx["retry_ok"] = True
        ctx["research_error"] = False
    else:
        ctx["retry_ok"] = False
        print(f"Research failed after {ctx['retry_count']} retries")


# Run the agent
agent = ResearchAgent()
agent.run(context={
    "topics": ["DataGrout competitors", "symbolic reasoning tools", "Prolog in production"],
    "domain": "AI tooling",
    "namespace": "research_findings",
    "analysis_goal": "What are the key differentiators and market gaps?",
    "report_title": "AI Tooling Landscape Analysis"
})
```

## Running with agents.orchestrate

```python
# Via the DG orchestrator (managed execution)
result = client.agents.orchestrate(
    agent_definition={
        "fsm": agent.to_dict(),
        "initial_context": {
            "topics": ["topic1", "topic2"],
            "namespace": "my_research"
        }
    },
    max_steps=50,
    timeout=300
)
print(result["final_context"]["report_ref"])
```

## See also

- [scheduled-monitoring](../../patterns/scheduled-monitoring/) — autonomous monitoring
- [research-to-report](../../recipes/combined/research-to-report/) — same pipeline as a single flow.into plan
