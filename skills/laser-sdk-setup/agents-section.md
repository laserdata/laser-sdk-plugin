## Laser SDK

This project uses the LaserData Laser SDK (release 0.8.0) over Apache Iggy.

- Connect with the env helper (`Laser.connect_env()` in Python, `Laser.connectEnv()` in TypeScript, `Laser::connect_env()` in Rust). It reads `LASER_CONNECTION_STRING` and optional `LASER_STREAM`. Never hard-code or print credentials.
- The connection string is `user:password@host[:port]`, port 8090 by default. `*.laserdata.cloud` and `*.laserdata.com` hosts get TLS automatically: add no TLS options for them.
- Name topics explicitly with `laser.stream(name).topic(name)`. Accessors do no I/O; `send`, `fetch`, `poll` and `ensure` do.
- Delivery is at least once and ordered per partition. Make handlers and external effects safe to repeat. Commit consumer offsets after the work succeeds.
- Query, key-value, change feed, graph and consumer filters need LaserData Cloud or Laser Stack, not plain Apache Iggy. Check `laser.capabilities()` first.
- Agent topics need `laser.bootstrap(partitions, retention)` once per stream before any agent starts. The SDK never calls a model: call it yourself and record it with `session.model(...)`.
- Durations are milliseconds. Absolute times are epoch microseconds.
- Do not guess API names. Check https://docs.laserdata.com/llms.txt, `foreign/python/laser_sdk.pyi` or `foreign/typescript/api/laser-sdk.api.md` in https://github.com/laserdata/laser-sdk at the installed version's tag, and `docs/parity.md` for the same call in another language.
