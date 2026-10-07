# mosqai-docs

The shared knowledge base for MosqAI Shield: architecture, contracts between
components, decisions, and the development workflow.

If two repositories have to agree on something (an MQTT topic, a REST payload, a
database meaning), the agreement lives **here** first, and code follows.

## Contents

| Document | What it answers |
| --- | --- |
| [architecture.md](architecture.md) | How the device, backend, AI, mobile and authority web fit together; sources of truth |
| [workflow.md](workflow.md) | Branching, commits, pull requests, and how the team works in parallel without conflicts |
| [roadmap.md](roadmap.md) | Milestones in order, and the issues each one breaks into |
| [contracts/mqtt.md](contracts/mqtt.md) | MQTT topics and payloads between ESP32 and backend (**draft — must be agreed by the core team**) |
| [decisions/](decisions/) | Architecture Decision Records (ADRs) |

## Contributing

Docs follow the same flow as code: branch from `develop`, open a PR, one review.
Changing a contract requires approval from every team it affects.

See [workflow.md](workflow.md).
