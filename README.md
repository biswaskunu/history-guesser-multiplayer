# history-guesser-multiplayer

An authoritative real-time multiplayer game server in Java, using Netty for networking.
The server owns game state; clients just send inputs and render what the server tells them.

## Docs

- [PRD](docs/PRD.md) — what we're building, requirements, success criteria.
- [ARCHITECTURE](docs/ARCHITECTURE.md) — Netty pipeline, packages, room state machine, concurrency.
- [WORKFLOW](docs/WORKFLOW.md) — solo + party game flows, edge cases, dev loop.
- [API](docs/API.md) — REST + WebSocket contract with examples.
- [MEMORY](docs/MEMORY.md) — locked decisions, Rust-parity rules, gotchas.
- [ROADMAP](docs/ROADMAP.md) — phased build plan with checkboxes.
