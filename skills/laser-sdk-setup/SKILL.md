---
name: laser-sdk-setup
description: "Add the LaserData Laser SDK to a project: install the package, set up the connection to LaserData Cloud, Laser Stack or Apache Iggy, add Laser SDK rules to the project's AGENTS.md, and run a first publish and read to prove it works. Use when the user says to set up, add, install or start using the Laser SDK or LaserData in a project."
---

# Set up the Laser SDK in a project

Work through these steps in order. Report what each step did. Stop and ask only when a choice is the user's to make.

## 1. Pick the language

Use the project's existing language: `pyproject.toml` or `requirements.txt` means Python, `package.json` means TypeScript, `Cargo.toml` means Rust. If there is none, ask.

## 2. Install

- Python: add `laser-sdk` to the project's dependencies (Python 3.10+).
- TypeScript: `npm install @laserdata/laser-sdk` (Node 22.14+, ESM only).
- Rust: `cargo add laser-sdk`, plus `--features agent` for agents and `--features managed` for query, KV and graph. Add `tokio` with `macros` and `rt-multi-thread`.

## 3. Choose where it connects

Ask which deployment unless the project already says:

- **LaserData Cloud.** If the `laser` CLI is installed and signed in, get the host from `laser deployment get --deployment-id <id> -o json` and the credentials from `laser deployment credentials --deployment-id <id> -o json`. Otherwise the user copies the connection string from the Console Credentials tab. TLS is automatic for LaserData hosts.
- **Laser Stack** (local, every feature): in a laser-stack checkout run `./scripts/up` and use the `LASER_CONNECTION_STRING` it prints.
- **Apache Iggy** (streaming and agents only). The 0.9.0 image refuses to start without an advertised address, and needs io_uring allowed:

  ```sh
  docker run -d --name iggy --security-opt seccomp=unconfined -p 8090:8090 \
    -e IGGY_TCP_ADDRESS=0.0.0.0:8090 \
    -e IGGY_NODE_ADVERTISED_ADDRESS=localhost \
    -e IGGY_SHARDING_CPU_ALLOCATION=all \
    -e IGGY_ROOT_USERNAME=iggy -e IGGY_ROOT_PASSWORD=iggy \
    apache/iggy:0.9.0
  ```

  The connection string is then `iggy:iggy@127.0.0.1:8090`. Use a different password for anything beyond a local test.

Put the value in `LASER_CONNECTION_STRING`, in the project's existing secrets mechanism (`.env` that is git-ignored, a secret store, or the shell). Never write a password into source, a committed file, or the chat. Check `.gitignore` covers any file that holds it.

## 4. Add the project rules

Append the Laser SDK section to the project's `AGENTS.md` (create it if missing) so every coding agent working in the repo follows the same rules. The text is in `agents-section.md` next to this skill. If the project has a `CLAUDE.md` and no `AGENTS.md`, add the section to `CLAUDE.md` instead. Do not duplicate it if a Laser SDK section is already there.

## 5. Prove it works

Write a small script in the project's language that connects with the env helper, ensures a topic in a scratch stream, publishes one JSON record, and reads it back with `replay().poll()`. The `laser-sdk` skill has the code for all three languages. Run it. Then print `await laser.capabilities()` so the user can see which managed features their deployment serves.

Delete the scratch stream afterwards with `laser.stream(name).delete()` if the user agrees.

## 6. Report

Tell the user the package and version installed, where it connects, which managed features are available, and the file the rules went into.
