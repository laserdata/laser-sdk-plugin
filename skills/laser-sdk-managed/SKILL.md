---
name: laser-sdk-managed
description: "Use the Laser SDK managed data features: key-value state with compare-and-swap and leases, queries over projections, the change feed, the knowledge graph, and server-side consumer filters, on LaserData Cloud or Laser Stack. Use when the user wants shared state between agents, an exactly-once effect, to query a topic like a table, to be notified when data changes, to filter records on the server, or to run these features locally. Load laser-sdk first for connecting."
---

# Laser SDK managed features

These features need LaserData Cloud or Laser Stack. Against plain Apache Iggy they fail with `UnsupportedError`. Check first:

- Python: `caps = await laser.capabilities()` then `caps.query.available`, `caps.kv.cas`
- TypeScript: `await laser.capabilities()`, plus `isOpenOnly` and `isReady` helpers

On a long-lived connection, `refresh_capabilities()` / `refreshCapabilities()` re-probes after a backend restart.

## Run them locally: Laser Stack

```sh
git clone https://github.com/laserdata/laser-stack
cd laser-stack
./scripts/up
```

`./scripts/up` prints a `LASER_CONNECTION_STRING` export. Laser Stack runs the Iggy fork and `laser-plane` in Docker and supports every SDK feature except Cloud-only UI.

The images use the `latest` tag, so a machine can hold stale ones. If managed calls fail with `TransportError: Invalid format` while `capabilities()` reports them available, the images are older than the SDK: run `docker compose pull` in the checkout and restart. SDK 0.8.0 works with `iggy-server` 0.9.4-ld and `laser-plane` 0.23.0.

## Key-value state

Python:

```python
kv = laser.kv("sessions")
await kv.set("user:42").json({"state": "online"}).ttl(300_000).send()
state = await kv.get_typed("user:42")
await kv.delete("user:42")
```

TypeScript (keys are bytes):

```ts
const key = new TextEncoder().encode("service:auth")
await laser.kv("config").set(key).json({ log_level: "debug" }).ttl(300_000).send()
const stored = await laser.kv("config").get(key)
```

**Make an effect happen once** with a compare-and-swap, for example `set(key)...expect_absent().commit()` in Rust. Leases (`kv.lease(key, holder, ttl_ms)`, then `renew_lease`, `release`) give one holder at a time. Fenced writes (`cas_fenced`) reject a replaced holder.

## Query a topic like a table

A projection turns a topic's records into an indexed view. Declare it once, during setup (Python shown, the dicts are the same shape in every SDK):

```python
await laser.projections().register({
    "id": "readings_v1.v1",
    "name": "readings_v1",
    "version": 1,
    "content_type": "json",
    "extraction": {
        "fields": [{"name": f, "pointer": f"/{f}"} for f in ("host_id", "cpu")],
        "inline_payload": False,
    },
    "inline_payload_default": False,
})
await laser.bindings().apply({
    "source": {"stream": "fleet", "topic": "readings"},
    "allowed_projections": ["readings_v1.v1"],
    "default_projection": "readings_v1.v1",
    "notify": True,  # publish a change feed for laser.watch()
    "index": "readings_v1",
})
```

The registration applies asynchronously: a query on the index raises until the table exists, then returns an empty page. Retry briefly before publishing records you expect to see. Rows also appear shortly after publishing, not instantly.

Query it:

Python:

```python
result = await (
    laser.query("readings_v1")
    .where_eq("host_id", "node-7")
    .filter_gte("cpu", 90)
    .order_desc("cpu")
    .limit(10)
    .fetch()
)
for row in result.rows:
    print(result.value_text(row, "host_id"), result.value(row, "cpu"))
```

`value()` returns a tagged value such as `{"kind": "long", "value": 95}`. Use `value_text()` for display.

TypeScript:

```ts
import { queryResultValue } from "@laserdata/laser-sdk"
const page = await laser.query("readings_v1").whereEq("status", "degraded").limit(20).fetch()
```

`fetch()` returns one page (at most 1000 rows) as a result you can read by field name. `fetch_all()` / `fetchAll()` follows the server cursor but returns bare rows with positional values and no field names. To read many pages by name, loop on `fetch()` and continue with `query.cursor(result.page.next_cursor)` while `result.page.has_more`. Each query has a 30-second deadline by default.

## Change feed

Wait for a view to advance instead of polling it:

```python
feed = laser.watch(index="readings_v1")
for change in await feed.poll():
    rows = await laser.query("readings_v1").fetch_all()  # the change is a wakeup, rows come from query
saved = feed.offsets  # persist to resume
```

TypeScript: `await laser.watch().index("readings_v1").records()`. Needs the `watch` capability.

## Knowledge graph

`laser.graph(name)` stores content-addressed nodes and edges. `neighbors(node, "out", depth=2)` in Python, `neighbors(node, "out", undefined, 2)` in TypeScript. Node and edge ids are the same in every SDK for the same content.

## Consumer filters

The server sends a consumer group only matching records. Configure the group once, then consume as normal:

```python
safe_mode = ls.ConsumerFilter.json(
    ls.FilterExpr.all([
        ls.FilterExpr.pred("table", "eq", "satellites"),
        ls.FilterExpr.pred("after.mode", "eq", "safe"),
    ])
)
group = laser.stream("orbit").topic("fleet_changes").consumer_group("anomaly-desk")
await group.create(filter=safe_mode)  # once, during setup

consumer = group.consumer(polling="first", auto_commit="disabled")
try:
    message = await consumer.next_within(15_000)  # milliseconds; TimeoutError when nothing matched
    await consumer.commit(message)
finally:
    await consumer.shutdown()
```

Repeating `create()` with the same filter succeeds, so it is safe in startup code. Change a filter with `group.filter().configure(...)`. An empty scan does not mean the end of the stream.

A filter saves bandwidth. It is not an access boundary. Use separate topics with separate grants to keep readers apart.

## Resource names

With a default stream, managed names (KV namespaces, memory, graph, projections, indexes) are scoped to it as `stream:<stream>/<name>`. A rerun on a recreated stream starts with empty managed state. `resource_naming="bare"` opts out.

## Permissions

Iggy permissions control stream and topic access. LaserData RBAC controls managed features, and a new user has no managed roles until one is bound. `laser.whoami()` shows the caller's effective capabilities. A denied managed call is a permissions problem, not a bug: say which capability is missing.

## More

- Key-value: https://docs.laserdata.com/laser-sdk/state
- Queries and views: https://docs.laserdata.com/laser-sdk/views
- Change feed: https://docs.laserdata.com/laser-sdk/change-feed
- Graph: https://docs.laserdata.com/laser-sdk/graph
- Filters: https://docs.laserdata.com/laser-sdk/consumer-filters
- Laser Stack: https://docs.laserdata.com/laser-sdk/laser-stack
- Examples: `examples/{python,typescript,rust}` `kv`, `query`, `watch`, `graph`, `cdc`
