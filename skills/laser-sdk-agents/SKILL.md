---
name: laser-sdk-agents
description: "Build AI agents with the Laser SDK: agent handlers, sessions, conversation context, memory, contracts between agents, and handler tests. Use when the user wants agents that message each other, record model and tool calls, keep state or memory across turns, hand work between agents, or replay a conversation, in Python, TypeScript or Rust on LaserData or Apache Iggy. Load laser-sdk first for connecting."
---

# Laser SDK agents

The agent runtime runs on plain Apache Iggy. It adds dedup, retry, dead letters, request and reply, routing by capability, sessions and context. **The SDK never calls a model.** The application calls its own model and records the call with the session helpers.

## Bootstrap the agent topics first

Every stream needs its agent topics before an agent joins. `agent.sessions` must have an explicit retention.

- Python: `await laser.bootstrap(4, retention=ls.TopicRetention.expire_after(86_400_000))`
- TypeScript: `await laser.bootstrap(4, TopicRetention.expireAfter(86_400_000))`
- Rust: `laser.bootstrap(4, TopicRetention::expire_after(Duration::from_secs(86_400))).await?`

If you will read sessions back through the managed session index (`laser.sessions().list()`, `get()`, `state()`, `events()`) on Laser Stack or Cloud, bootstrap with `laser.sessions().bootstrap(4, retention)` instead. It creates the same topics and also registers the stream as a session source.

Give each application, and each environment, its own stream. The stream is the isolation boundary.

## Spawn an agent

Python:

```python
import laser_sdk as ls

async def handle(ctx, message):
    text = message.body().decode()
    await ctx.respond(f"echo: {text}".encode())

agent = laser.spawn_agent(
    "triage",
    ls.AgentTopic.Sessions,
    handle,
    respond_on=ls.AgentTopic.Sessions,
    capabilities=["resolve-ticket"],
)
await agent.ready()
# ... later
await agent.shutdown()
```

TypeScript:

```ts
import { Agent, AgentId, AgentTopic } from "@laserdata/laser-sdk"

await using agent = Agent.builder()
  .id(AgentId.new("triage"))
  .listenOn(AgentTopic.Sessions)
  .respondOn(AgentTopic.Sessions)
  .capabilities([{ skillId: "resolve-ticket" }])
  .handler({ handle: (message, ctx) => ctx.respond(message.payload) })
  .build()
  .spawn(laser)
await agent.ready()
```

Rust (feature `agent`, `use laser_sdk::prelude::full::*;`):

```rust
struct Triage;

impl AgentHandler for Triage {
    async fn handle(&self, message: &AgentMessage, ctx: &AgentCtx<'_>) -> Result<(), LaserError> {
        ctx.respond("on it").await
    }
}

let mut triage = Agent::builder()
    .id("triage".parse()?)
    .listen_on(AgentTopic::Sessions)
    .respond_on(AgentTopic::Sessions)
    .handler(Triage)
    .build()
    .spawn(laser.clone());
triage.ready().await?;
```

Each agent id reads through its own consumer group, so running more instances of one agent shares its work. The handler sees commands and plain records addressed to it or to every agent, including commands the agent's own session writes. Check `message.envelope["operation"]`, or pass `operations=[...]` to limit what the handler serves (see Hand work to an agent). Offsets commit after the handler succeeds, so a crash means redelivery: keep handlers idempotent.

## Ask an agent by capability (contract)

A contract is a task with a deadline and a definite outcome: completed, failed, not consumed, or timed out.

Python:

```python
reply = await laser.contract(
    "resolve-ticket",
    b"ticket #42 is stuck",
    source="orchestrator",
    fixed_inbox=ls.AgentTopic.Sessions,
    deadline_ms=60_000,
)
if isinstance(reply, ls.Contract.Completed):
    print(bytes(reply[0].body()).decode())
```

TypeScript:

```ts
import { routeToCapable } from "@laserdata/laser-sdk"

const outcome = await laser
  .contract(routeToCapable("resolve-ticket", { kind: "any" }))
  .from(AgentId.new("orchestrator"))
  .payload(new TextEncoder().encode("ticket #42 is stuck"))
  .inboxRoute({ kind: "fixed", topic: AgentTopic.Sessions })
  .deadline(60_000)
  .send()
if (outcome.kind === "completed") { /* outcome.reply */ }
```

Inside a handler, `ctx.fan_out(skill, body, deadline_ms=...)` asks every capable agent, and `ctx.approval_gate(topic, prompt, timeout_ms=...)` pauses for a human.

## Sessions: one unit of work, recorded

A session is one conversation with a lifecycle on `agent.sessions`. Record each model and tool call on it so the work can be inspected and replayed.

Python:

```python
sessions = laser.sessions()
async with sessions.create("ticket-42").agent("triage") as session:
    assembled = await session.assemble(ls.LastN(20))
    call = await session.model(ls.ModelRequest("gpt-4o", assembled.text(), provider="openai"), assembled)
    answer = await my_model(assembled.text())  # your model call
    await call.complete(ls.ModelResponse(answer))
    tool = await session.tool("lookup", {"ticket": 42})
    await tool.complete(b"found")
    await session.state().set("status", "triaged")
# The block ends the session, or fails it if it raised.
```

TypeScript:

```ts
const { session, lease } = await laser.sessions().create("ticket-42").agent(AgentId.new("triage")).begin()
await session.run(lease, async () => {
  const call = await session.model(new ModelRequest("gpt-4o", new TextEncoder().encode(prompt)))
  await call.complete({ body: new TextEncoder().encode(answer) })
  await session.state().set("status", "triaged")
})
```

`create(label)` gives a stable id for the same label. `start()` makes a fresh one. Tool arguments are redacted by default (`api_key` and similar).

### Hand work to an agent

Submit a session to an agent and the worker's handler reaches it with `ctx.session()`. **Spawn the worker with `operations=["invoke_agent"]`** (TypeScript `.operations(["invoke_agent"])`). A submitted session arrives as an `invoke_agent` command, but `session.tool(...)` called inside the handler writes an `execute_tool` command addressed to the same agent. Without the `operations` limit the handler receives its own tool calls as new work, fails to parse them or loops.

```python
async def handle(ctx, message):
    ticket = json.loads(bytes(message.body()))
    session = ctx.session()
    tool = await session.tool("classify", {"subject": ticket["subject"]})
    await tool.complete(b"P2")
    await session.state().set("status", "triaged")
    await session.end()

worker = laser.spawn_agent("triage", ls.AgentTopic.Sessions, handle, operations=["invoke_agent"])
await worker.ready()
await laser.sessions().submit("triage", json.dumps(ticket).encode()).from_("intake").send()
```

TypeScript spells the submitter `.from(AgentId.new("intake"))`.

Read back: `session.context()` returns the turns, `session.checkpoint()` returns a serialisable position, and `session.turns_since(checkpoint)` resumes from it.

## Context for a prompt

`laser.context(conversation)` returns a scope over one conversation. It takes a conversation id, not a label: for a session, pass `session.conversation`.

- Python: `turns = await ctx.fetch(n=20, token_budget=4_000)`
- TypeScript: `await ctx.fetch([AgentTopic.Sessions], 20, 4_000)` (topics, count, token budget)

Tokens are estimated as bytes divided by four, rounded up. Ordering across topics is best effort by timestamp.

## Memory

`laser.memory(namespace)` is log-backed memory with `remember`, `recall`, `forget` and `improve`. Writes work on Apache Iggy. The default `recall` reads a managed view and needs Laser Stack or LaserData Cloud. On plain Iggy use `recall(folded=True)` (Python) or `.recall().folded().fetch()` (TypeScript) to build the view locally.

For semantic recall, supply an embedder: Python `VectorMemory(embed)` or `laser.memory_with(namespace, "vector", embedder=embed)`. The SDK ships no embedding model.

Named working state: `laser.memory("notes").set("plan", json.dumps(plan))` and `.fetch("plan")`, which returns bytes. Values are `str` or `bytes`, not dicts. On Laser Stack and Cloud the read view lags a write slightly, so a `fetch` straight after `set` can return `None`. Use `fetch_folded` to read from the log directly.

## Test a handler without a server

- Python: `from laser_sdk import agent_message, agent_ctx, Provenance`, then `message = agent_message(b"hello", Provenance(agent="tester"))` and `ctx = agent_ctx(laser, message, agent="tester", respond_on="agent.sessions")`, and call the handler directly.
- TypeScript: `agentMessage` and `agentCtx` from `@laserdata/laser-sdk/testing`.

## Gotchas

- On plain Apache Iggy there is no live presence, so agents cannot advertise an inbox (the SDK logs "inbox presence not advertised"). Route contracts to a fixed topic, as in the examples above (`fixed_inbox=` / `inboxRoute({ kind: "fixed", ... })`). Laser Stack and LaserData Cloud add discovery.
- One connection can advertise one agent. Give each capability-advertising agent its own connection, especially in tests.
- `agent.control` is operator-only and is never created by `bootstrap`.
- Session budgets are cooperative limits, not hard spending caps.
- Python handlers are `async def handle(ctx, message)`. TypeScript handlers are `handle(message, ctx)`. Note the argument order.

## More

- Agents: https://docs.laserdata.com/laser-sdk/fabric
- Sessions: https://docs.laserdata.com/laser-sdk/session
- Memory: https://docs.laserdata.com/laser-sdk/memory
- Context: https://docs.laserdata.com/laser-sdk/context
- Worked scenario: `docs/building-agents.md` in https://github.com/laserdata/laser-sdk
- Examples: `examples/{python,typescript,rust}` `agent`, `sessions`, `recall`, `context`, `orchestra`
