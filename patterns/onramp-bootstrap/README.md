# Pattern: Onramp and Bootstrap

> **Difficulty:** Intermediate  
> **Credits:** 0  
> **Time:** ~10 minutes

## What it does

Connect an autonomous agent to DataGrout without a human registration step. The onramp flow issues provisional OAuth credentials via two plain HTTP POST requests, no MCP client, no browser, no API key pre-configured. Bootstrap then derives mTLS identity from those credentials so all subsequent calls are mutually authenticated.

## Two paths

| Path | When to use |
|------|-------------|
| `bootstrap_onramp` | First time an agent connects — no credentials exist yet |
| `Client(identity_auto=True)` | Agent has already onboarded — cert lives in `~/.conduit/` |

Start with `bootstrap_onramp`. On every run after, use `identity_auto`.

## The onramp protocol

```
POST /onramp            { agent_name, agent_type, intended_use }
← 200 { session_token }   (5-minute TTL, prevents fire-and-forget bulk registration)

POST /onramp/complete   Authorization: Bearer <session_token>
← 200 { client_id, client_secret, token_url, rpc_url, mcp_url }
```

The two-step handshake proves the caller is present and listening. The returned `client_id` + `client_secret` are provisional OAuth credentials with restricted scopes.

## Python: one-shot bootstrap

```python
import asyncio
from datagrout.conduit import Client
from datagrout.conduit.onramp import OnrampOptions

async def main():
    # First run: register the agent and get mTLS identity in one call.
    client = await Client.bootstrap_onramp(
        OnrampOptions(
            gateway="https://app.datagrout.ai",
            agent_name="my-research-agent",
            agent_type="claude-sonnet-4-6",
            intended_use="Summarise documents and extract entities.",
        )
    )
    await client.connect()

    result = await client.perform("data-grout@1/prism.analyze@1", {
        "goal": "Summarise this paragraph",
        "data": "DataGrout reduces LLM token usage through symbolic reasoning."
    })
    print(result["answer"])

asyncio.run(main())
```

On subsequent runs, the mTLS cert is on disk, use `identity_auto=True` instead:

```python
# Subsequent runs: cert already in ~/.conduit/: no onramp needed.
client = Client(
    url="https://app.datagrout.ai/servers/<uuid>/mcp",
    identity_auto=True,
)
```

## TypeScript: one-shot bootstrap

```typescript
import { Client } from '@datagrout/conduit';

// First run: register and bootstrap.
const client = await Client.bootstrapOnramp({
  opts: {
    gateway: 'https://app.datagrout.ai',
    agentName: 'my-research-agent',
    agentType: 'claude-sonnet-4-6',
    intendedUse: 'Summarise documents and extract entities.',
  },
});
await client.connect();

const result = await client.perform('data-grout@1/prism.analyze@1', {
  goal: 'Summarise this paragraph',
  data: 'DataGrout reduces LLM token usage through symbolic reasoning.',
});
console.log(result.answer);
```

Subsequent runs:
```typescript
// Cert discovered from ~/.conduit/ automatically.
const client = new Client({
  url: 'https://app.datagrout.ai/servers/<uuid>/mcp',
  identityAuto: true,
});
```

## Low-level: step by step

Use `register_only` or `register_and_exchange` when you need to control storage or persist credentials yourself:

```python
from datagrout.conduit.onramp import register_only, register_and_exchange, OnrampOptions

opts = OnrampOptions(
    gateway="https://app.datagrout.ai",
    agent_name="pipeline-worker",
    agent_type="claude-haiku-4-5-20251001",
    intended_use="Classify incoming support tickets.",
)

# Step 1: get provisional credentials
creds = await register_only(opts)
print(creds.client_id)     # persist this
print(creds.client_secret) # store securely, shown once

# Step 2: exchange for an access token
creds, access_token = await register_and_exchange(opts)

# Step 3: build a client with those credentials
client = Client(
    url=creds.rpc_url,
    auth={"client_credentials": {
        "client_id": creds.client_id,
        "client_secret": creds.client_secret,
        "token_endpoint": creds.token_url,
    }}
)
```

## Rust: bootstrap_onramp

```rust
use datagrout_conduit::{Client, OnrampOptions};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::bootstrap_onramp(OnrampOptions {
        gateway: "https://app.datagrout.ai".into(),
        agent_name: "rust-agent".into(),
        agent_type: Some("claude-sonnet-4-6".into()),
        intended_use: Some("Process and classify documents.".into()),
        ..Default::default()
    })
    .await?;

    client.connect().await?;

    let result = client.perform("data-grout@1/prism.analyze@1", serde_json::json!({
        "goal": "Summarise this text",
        "data": "DataGrout symbolic reasoning."
    })).await?;

    println!("{}", result["answer"]);
    Ok(())
}
```

## Elixir: bootstrap_onramp

```elixir
{:ok, client} = DatagroutConduit.Client.bootstrap_onramp(%{
  gateway: "https://app.datagrout.ai",
  agent_name: "elixir-agent",
  agent_type: "claude-sonnet-4-6",
  intended_use: "Extract entities from documents."
})

{:ok, _} = DatagroutConduit.connect(client)

{:ok, result} = DatagroutConduit.call(client, "data-grout@1/prism.analyze@1", %{
  goal: "Summarise this text",
  data: "DataGrout symbolic reasoning."
})
IO.inspect(result["answer"])
```

## Ruby: bootstrap_onramp

```ruby
require 'datagrout_conduit'

client = DatagroutConduit::Client.bootstrap_onramp(
  gateway: 'https://app.datagrout.ai',
  agent_name: 'ruby-agent',
  agent_type: 'claude-sonnet-4-6',
  intended_use: 'Summarise and classify documents.'
)

result = client.perform('data-grout@1/prism.analyze@1', {
  goal: 'Summarise this text',
  data: 'DataGrout symbolic reasoning.'
})
puts result['answer']
```

## Security notes

- `client_secret` is shown **exactly once** — persist it to a secret store (AWS Secrets Manager, Vault, etc.) immediately
- Provisional credentials have restricted scopes; the server owner can grant elevated scopes via `access_code`
- mTLS certs written to `~/.conduit/` are rotated automatically before expiry when `identity_auto=True`

## See also

- [jsonrpc-transport](../jsonrpc-transport/) — raw JSONRPC and WebSocket transport options
- [hello-world](../../quickstart/hello-world/) — connect with a plain API key (no onramp)
