# Pattern: JSONRPC and WebSocket Transport

> **Difficulty:** Intermediate  
> **Credits:** 0  
> **Time:** ~10 minutes

## What it does

The Conduit SDK ships three transports. Understand when to use each one and how to configure them:

| Transport | Protocol | Best for |
|-----------|----------|----------|
| `mcp` (default) | HTTP + MCP envelope | Claude Code, MCP-aware agents |
| `jsonrpc` | HTTP + JSON-RPC 2.0 | Server-side services, non-MCP environments |
| `websocket` | WS + JSON-RPC 2.0 | Real-time subscriptions, agent event streams |

The default transport is `mcp`. Switch to `jsonrpc` or `websocket` by passing `transport=` at construction time. All three expose the same `perform` / `call` API, the transport is transparent to your tool calls.

## MCP transport (default)

```python
from datagrout.conduit import Client

# Default: MCP transport, used automatically by Claude Code
client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

result = client.perform("data-grout@1/prism.analyze@1", {
    "goal": "Summarise this text",
    "data": "DataGrout reduces LLM token usage through symbolic reasoning."
})
```

When connected via Claude Code (`claude mcp add datagrout`), the SDK resolves the MCP server URL from the Claude config automatically.

## JSONRPC transport

Use the JSONRPC transport when:
- Your code runs outside Claude (server-side pipeline, batch job, microservice)
- You want lower overhead than MCP (no tool envelope wrapping/unwrapping)
- You're behind a network that restricts MCP but allows plain HTTPS

```python
from datagrout.conduit import Client

client = Client(
    url="https://app.datagrout.ai/servers/<uuid>/rpc",
    transport="jsonrpc",
    auth={"bearer": "your-token"},
)

result = await client.perform("data-grout@1/prism.analyze@1", {
    "goal": "Classify this support ticket by urgency",
    "data": ticket_text,
})
```

### TypeScript

```typescript
import { Client } from '@datagrout/conduit';

const client = new Client({
  url: 'https://app.datagrout.ai/servers/<uuid>/rpc',
  transport: 'jsonrpc',
  auth: { bearer: 'your-token' },
});

const result = await client.perform('data-grout@1/prism.analyze@1', {
  goal: 'Classify this ticket',
  data: ticketText,
});
```

### With mTLS (after onramp)

```python
from datagrout.conduit import Client
from datagrout.conduit.identity import ConduitIdentity

identity = ConduitIdentity.try_default()  # reads from ~/.conduit/

client = Client(
    url="https://app.datagrout.ai/servers/<uuid>/rpc",
    transport="jsonrpc",
    identity=identity,
)
```

### With OAuth client credentials

```python
client = Client(
    url="https://app.datagrout.ai/servers/<uuid>/rpc",
    transport="jsonrpc",
    auth={
        "client_credentials": {
            "client_id": "your-client-id",
            "client_secret": "your-client-secret",
            "token_endpoint": "https://app.datagrout.ai/oauth/token",
        }
    }
)
# Token is fetched and refreshed automatically per request.
```

## WebSocket transport

Use WebSockets when you need server-pushed notifications, task progress, agent events, log streaming. A single `wss://` connection multiplexes all in-flight requests with no head-of-line blocking.

Wire protocol: `datagrout-jsonrpc.v1` subprotocol, text frames only (JSON-RPC 2.0).

```
wss://app.datagrout.ai/servers/<uuid>/ws
```

### Python: basic

```python
from datagrout.conduit import Client

async with Client(
    url="wss://app.datagrout.ai/servers/<uuid>/ws",
    transport="websocket",
    auth={"bearer": "your-token"},
) as client:
    result = await client.perform("data-grout@1/prism.analyze@1", {
        "goal": "Summarise this",
        "data": "Some text to summarise."
    })
    print(result["answer"])
```

### Python: subscriptions (server-push)

```python
async with Client(url="wss://...", transport="websocket", auth=...) as client:
    # Subscribe to agent events
    sub = await client.subscribe("agents.my-agent-id.events")

    async for event in sub:
        print(f"{event.event}: {event.data}")
        if event.event == "agent.completed":
            break

    await client.unsubscribe(sub.id)
```

### Real-time log stream

```python
async with Client(url="wss://...", transport="websocket", auth=...) as client:
    sub = await client.subscribe("log.stream")

    async for event in sub:
        log_line = event.data
        if log_line.get("level") == "error":
            # route to forensic investigation
            await client.perform("data-grout@1/forensic.investigate@1", {
                "data": log_line,
                "goal": "Root cause of this error"
            })
```

### TypeScript: subscriptions

```typescript
import { Client } from '@datagrout/conduit';

const client = new Client({
  url: 'wss://app.datagrout.ai/servers/<uuid>/ws',
  transport: 'websocket',
  auth: { bearer: 'your-token' },
});

await client.connect();

const sub = await client.subscribe('agents.my-agent-id.events');

for await (const event of sub) {
  console.log(event.event, event.data);
  if (event.event === 'agent.completed') break;
}

await client.unsubscribe(sub.id);
await client.disconnect();
```

## Transport comparison

```
MCP transport:
  URL:        https://.../mcp
  Auth:       API key header or mTLS
  Envelope:   MCP content array (unwrapped transparently by SDK)
  Use when:   Claude Code, MCP agents, tool calls from the Claude UI

JSONRPC transport:
  URL:        https://.../rpc
  Auth:       Bearer, Basic, OAuth client credentials, mTLS
  Envelope:   JSON-RPC 2.0 ({"jsonrpc":"2.0","id":1,"method":"tools/call",...})
  Use when:   Server processes, batch pipelines, non-MCP environments

WebSocket transport:
  URL:        wss://.../ws
  Auth:       Bearer or API key in upgrade headers, mTLS
  Subproto:   datagrout-jsonrpc.v1
  Use when:   Agent event streams, task progress, log tailing, real-time routing
```

## Reconnection

The WebSocket transport does not auto-reconnect. After a disconnect, `send_request` raises `RuntimeError("WS transport not connected")`. Active subscriptions do not survive reconnects, re-subscribe after calling `connect()` again.

```python
async def connect_with_retry(client, max_retries=3):
    for attempt in range(max_retries):
        try:
            await client.connect()
            return
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            await asyncio.sleep(2 ** attempt)
```

## Install WebSocket extras

The WebSocket transport requires the `websockets` package, which is not installed by default:

```bash
pip install 'datagrout-conduit[ws]'
npm install @datagrout/conduit   # ws included in the npm package
cargo add datagrout-conduit      # feature: ws
```

## See also

- [onramp-bootstrap](../onramp-bootstrap/) — how to get credentials before connecting
- [background-tasks](../background-tasks/) — WebSocket task progress monitoring
- [log-processing](../../recipes/flow/log-processing/) — real-time log routing example
