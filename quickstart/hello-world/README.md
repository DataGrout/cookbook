# Recipe: Hello World

## What it does

Connect to DataGrout, run `discovery.summary` to see what's available, then call one tool. Three steps, minimal code.

## Tools used

- `discovery.summary` — overview of all available tools
- `discovery.discover` — semantic search for a specific tool

## Credits cost

2 credits total (1 per discovery call).

## The Recipe

### Python

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

# Step 1: See what's available
summary = client.discovery.summary()
print(summary["text"])

# Step 2: Find a specific tool
results = client.discovery.discover(goal="analyze data")
for r in results["results"]:
    print(f"{r['tool_name']}: {r['score']:.2f}")

# Step 3: Call a tool
analysis = client.prism.analyze(
    goal="What are the key themes in this text?",
    data="DataGrout reduces LLM token usage by 10-100x through symbolic reasoning."
)
print(analysis["result"])
```

### TypeScript

```typescript
import { Client } from '@datagrout/conduit';

const client = new Client({ apiKey: 'your-key' });

const summary = await client.discovery.summary();
console.log(summary.text);

const analysis = await client.prism.analyze({
  goal: 'What are the key themes in this text?',
  data: 'DataGrout reduces LLM token usage by 10-100x through symbolic reasoning.'
});
console.log(analysis.result);
```

### Elixir

```elixir
{:ok, client} = DatagroutConduit.Client.start_link(api_key: "your-key")

{:ok, summary} = DatagroutConduit.call(client, "discovery.summary", %{})
IO.puts(summary["text"])

{:ok, analysis} = DatagroutConduit.call(client, "prism.analyze", %{
  goal: "What are the key themes in this text?",
  data: "DataGrout reduces LLM token usage by 10-100x through symbolic reasoning."
})
IO.inspect(analysis["result"])
```

### Ruby

```ruby
require 'datagrout_conduit'

client = DatagroutConduit::Client.new(api_key: 'your-key')

summary = client.call('discovery.summary', {})
puts summary['text']

analysis = client.call('prism.analyze', {
  goal: 'What are the key themes in this text?',
  data: 'DataGrout reduces LLM token usage by 10-100x through symbolic reasoning.'
})
puts analysis['result']
```

### Rust

```rust
use datagrout_conduit::Client;

#[tokio::main]
async fn main() {
    let client = Client::new("your-key");

    let summary = client.call("discovery.summary", serde_json::json!({})).await.unwrap();
    println!("{}", summary["text"]);

    let analysis = client.call("prism.analyze", serde_json::json!({
        "goal": "What are the key themes in this text?",
        "data": "DataGrout reduces LLM token usage by 10-100x through symbolic reasoning."
    })).await.unwrap();
    println!("{}", analysis["result"]);
}
```

## Try it

```bash
# Python
pip install datagrout-conduit
python hello_world.py

# TypeScript
npm install @datagrout/conduit
npx ts-node hello_world.ts
```

## Next steps

- [first-flow](../first-flow/) — compose two tools with `flow.into`
- [first-logic-cell](../first-logic-cell/) — assert facts and query them
