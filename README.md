# Laser SDK plugin

Skills that teach coding agents to build with the [LaserData Laser SDK](https://github.com/laserdata/laser-sdk) in Python, TypeScript and Rust. One package installs in Claude Code, Cursor and Codex.

| Skill | Use it for |
| --- | --- |
| `laser-sdk` | Connecting, publishing, consuming, typed records, errors. Loads first. |
| `laser-sdk-agents` | Agents, contracts, sessions, context, memory, handler tests |
| `laser-sdk-managed` | Key-value, queries, change feed, graph, consumer filters, Laser Stack |
| `laser-sdk-setup` | Adding the SDK to a project and writing its AGENTS.md section |

The skills match Laser SDK 0.8.0. No server or sign-in is needed: the plugin is instructions only. For deployments and credentials, pair it with the LaserData CLI skills in [laserdata/laser-cli-claude](https://github.com/laserdata/laser-cli-claude).

## Install

Claude Code:

```text
/plugin marketplace add laserdata/laser-sdk-plugin
/plugin install laser-sdk@laser-sdk
```

From a shell, the same is `claude plugin marketplace add laserdata/laser-sdk-plugin` then `claude plugin install laser-sdk@laser-sdk`.

Cursor and Codex: install `laser-sdk` from the plugin marketplace once it is listed. Until then, Codex picks the plugin up from `.agents/plugins/marketplace.json` in a clone of this repository.

Any other agent: copy `skills/laser-sdk-setup/agents-section.md` into the project's `AGENTS.md`.

## Layout

```
plugin.json                       portable manifest (Cursor, Codex)
.claude-plugin/plugin.json        Claude Code manifest
.claude-plugin/marketplace.json   Claude Code marketplace, this repo as one plugin
.agents/plugins/marketplace.json  Codex marketplace
skills/<name>/SKILL.md            the skills
```

## Checks

`claude plugin validate --strict .` passes. The snippets were run against Apache Iggy 0.9.0 with laser-sdk 0.8.0 from PyPI, npm and crates.io. Python and TypeScript covered publish, replay, consumer groups, agents, contracts, sessions, context and the handler test helpers. Rust covered publish, replay and spawning an agent. The managed Python snippets (key-value, projection and query, change feed, named memory, consumer filters, whoami) were run against Laser Stack with `iggy-server` 0.9.4-ld and `laser-plane` 0.23.0. The graph snippets have not been run.

Licensed under Apache-2.0.
