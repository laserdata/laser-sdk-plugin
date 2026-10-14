---
name: laser-sdk
description: "Build with the LaserData Laser SDK (Python, TypeScript or Rust) over Apache Iggy, LaserData Cloud or Laser Stack. Use when code imports laser_sdk, @laserdata/laser-sdk or the laser-sdk crate, when the user wants to publish or consume messages, connect to a LaserData deployment, or build agents, memory, sessions, queries or key-value state on LaserData. Load this first, then laser-sdk-agents or laser-sdk-managed for those areas."
---

# Laser SDK

Laser SDK is one client over Apache Iggy for streaming, agents, memory, sessions, queries and key-value state. The log is the source of truth. Everything else is a view built from it. These notes match release 0.8.0 of all three SDKs. Before 1.0.0 a minor release can break the API, so check the installed version first.

## Pick the right skill

- Connecting, publishing, consuming, typed records, errors: this skill.
- Agents, sessions, context, memory, contracts, handler tests: `laser-sdk-agents`.
- Key-value, queries, change feed, graph, consumer filters, Laser Stack: `laser-sdk-managed`.
- Adding the SDK to a project for the first time: `laser-sdk-setup`.

## Where it runs

| Deployment | What works |
| --- | --- |
| Apache Iggy | Streaming, the agent runtime, sessions, log-backed memory, contracts, workflows |
| Laser Stack (local Docker) | All of the above, plus consumer filters, query, KV, forks, graph, durable memory, the session index |
| LaserData Cloud | Everything, with managed deployments |

A managed call against plain Apache Iggy fails with `UnsupportedError` (Python, TypeScript) or `LaserError::Unsupported` (Rust). Check `laser.capabilities()` before using a managed feature, and tell the user which deployment it needs instead of letting the error surface raw.

## Install

| Language | Command | Notes |
| --- | --- | --- |
| Python | `pip install laser-sdk` | Python 3.10+, wheels for Linux and macOS. Import `laser_sdk`. |
| TypeScript | `npm install @laserdata/laser-sdk` | Node 22.14+. ESM. No Bun, Deno or browser. |
| Rust | `cargo add laser-sdk` | Default features: `streaming`, `provenance`. Add `--features agent` and/or `managed`. |

## Connect

The connection string is `user:password@host[:port]` or `token@host[:port]`. The port defaults to 8090. The scheme is optional and may only be `iggy://` or `iggy+tcp://`. Credentials are required.

Never hard-code credentials. Read `LASER_CONNECTION_STRING` (and optional `LASER_STREAM`) with the env helper:

- Python: `await Laser.connect_env()`, or `await Laser.local()` for `iggy:iggy@127.0.0.1:8090`
- TypeScript: `await Laser.connectEnv()`, `await Laser.local()`
- Rust: `Laser::connect_env().await?`, `Laser::local().await?`

Hosts under `*.laserdata.cloud` and `*.laserdata.com` get TLS automatically with the CA bundled in the SDK. Do not add TLS options for them. `LASER_TLS_CERT=<path>` sets a CA for any other host. `LASER_NO_TLS=1` turns automatic TLS off.

For a LaserData Cloud deployment, the username and password come from the Console Credentials tab or `laser deployment credentials --deployment-id <id> -o json`, and the host from `laser deployment get`. Never echo the password into chat or write it to a file the user did not name. Use a placeholder such as `<password>` in snippets.

Connecting gives up after 30 seconds (`LASER_CONNECT_TIMEOUT_MS` changes it).

## The grammar

An accessor picks a feature and does no I/O. A terminal verb (`send`, `fetch`, `poll`, `ensure`) does the work. `laser.stream(name).topic(name)` names a topic. `laser.topic(name)` works only when the connection has a default stream (`connect_with_stream` / `connectWithStream`, or `LASER_STREAM`). A default stream does not restrict access to other streams.

## Publish and read back

Python:

```python
import asyncio
from laser_sdk import Laser

async def main():
    laser = await Laser.connect_env()
    topic = laser.stream("telemetry").topic("inferences")
    await topic.ensure(partitions=4)
    await topic.publish().json({"latency_ms": 42}).send()

    cursor = topic.replay()
    for message in await cursor.poll():
        print(message.json())
    saved = cursor.offsets  # persist to resume after a restart
    await laser.close()

asyncio.run(main())
```

TypeScript:

```ts
import { Laser } from "@laserdata/laser-sdk"

await using laser = await Laser.connectEnv()
const topic = laser.stream("telemetry").topic("inferences")
await topic.ensure(4)
await topic.publish().json({ latency_ms: 42 }).send()

const records = await (await topic.replay()).poll()
console.log(`read ${records.length} record(s)`)
```

Rust:

```rust
use laser_sdk::prelude::*;

#[tokio::main]
async fn main() -> Result<(), LaserError> {
    let laser = Laser::connect_env().await?;
    let topic = laser.stream("telemetry").topic("inferences");
    topic.ensure(4).await?;
    topic.publish().json(&serde_json::json!({ "latency_ms": 42 }))?.send().await?;

    let messages = topic.replay()?.poll().await?;
    println!("read {} message(s)", messages.len());
    Ok(())
}
```

`await using` needs TypeScript compilation, or Node 24 for plain JavaScript. On Node 22 in plain `.mjs`, call `await laser.close()` instead.

Payloads are bytes in TypeScript (`Uint8Array`, use `TextEncoder`/`TextDecoder`). Python returns `bytes`.

## Live consumers

Use a consumer group for continuous processing. Turn automatic commits off and commit after the work succeeds, or a crash can skip records.

Python:

```python
consumer = topic.consumer_group("workers").consumer(auto_commit="disabled")
await consumer.init()
try:
    message = await consumer.next()
    if message is not None:
        await handle(message.payload, message.headers)
        await consumer.commit(message)
finally:
    await consumer.shutdown()
```

TypeScript:

```ts
const consumer = await topic.consumerGroup("workers").consumer({ commitPolicy: { kind: "disabled" } })
try {
  const record = await consumer.nextWithin(15_000) // TimeoutError when nothing arrives
  await handle(record)
  await consumer.commit(record)
} finally {
  await consumer.shutdown()
}
```

A `replay()` cursor reads retained records and stops when caught up. A consumer waits for new ones. Every instance in one group shares the partitions, so instances beyond the partition count sit idle.

## Typed records

- Python: `laser.stream("fleet").topic("readings").json(Reading)` binds a dataclass or pydantic model. `records(reader_name)` reads them back and raises `TypedDecodeError` with the log position on a bad record.
- TypeScript: types do not exist at runtime, so pass a checking codec: `topic.json(new Json<Reading>(check))`.
- Rust: `.json::<T>()`, `.cbor::<T>()`, or `.schema::<T>(id)` for a registered Avro, Protobuf or JSON Schema.

## Rules that bite

- Delivery is at least once. Handlers and external effects must be safe to repeat. For an effect that must happen once, use a KV compare-and-swap (see `laser-sdk-managed`).
- Ordering holds within a partition only. Use a key (`partition_key` / `key`) to keep related records together.
- A publish retries three times with backoff, then fails with `PublishFailedError` listing `committed` and `unconfirmed` records. Unconfirmed records may already be on the server, so check before resending.
- All relative durations are milliseconds (`timeout_ms`, `ttl_ms`). Absolute times are epoch microseconds.
- Close what you open: `await laser.close()` in Python, `await using` in TypeScript.
- Errors carry classifiers so you can branch without matching types: Python attributes such as `retryable`, `unsupported`, `permission_denied`. TypeScript functions such as `isRetryable`, `isUnsupported`, `isPermissionDenied`.

## Look things up, do not guess

The API is large. When a call is not shown here, check the source of truth rather than inventing a name:

- Docs index for agents: https://docs.laserdata.com/llms.txt (full text: `/llms-full.txt`)
- Python signatures: `foreign/python/laser_sdk.pyi` in https://github.com/laserdata/laser-sdk
- TypeScript signatures: `foreign/typescript/api/laser-sdk.api.md` in the same repo
- Rust to Python to TypeScript name map: `docs/parity.md`
- Runnable examples in all three languages: `examples/`

Read the version that matches the installed package (tags such as `py-v0.8.0`, `ts-v0.8.0`, `rust-v0.8.0`).
