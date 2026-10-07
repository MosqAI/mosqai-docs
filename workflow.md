# Development workflow

## Branches

| Branch | Purpose |
| --- | --- |
| `main` | Released, deployable code. Changed only by merging `develop` (or a hotfix) via PR. |
| `develop` | Integration branch. All feature work merges here via PR. |
| `feature/<short-name>` | New work, e.g. `feature/device-heartbeat` |
| `fix/<short-name>` | Bug fixes, e.g. `fix/device-offline-state` |
| `refactor/<short-name>` | Behaviour-preserving restructuring |

Branch from `develop`, keep branches short-lived (a few days), and rebase on
`develop` before opening a PR.

## Commits

[Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add device registration
fix: handle device heartbeat timeout
docs: document MQTT topics
test: cover command acknowledgement timeout
refactor: extract device state mapper
chore: bump prisma to 6.x
```

## Pull requests

- Every change goes through a PR. Nothing is pushed directly to `main` or `develop`.
- One approving review from the other developer. Contract changes need both.
- CI must pass (lint, type-check, tests) once CI exists for that repo.
- Link the issue: `Closes #12`.
- Squash-merge into `develop`; merge-commit `develop` → `main` for releases.

## Working in parallel without conflicts

1. **Contracts first.** Before Developer 2 writes MQTT code in firmware and either
   developer writes it in the backend, the topic/payload is agreed in
   `mosqai-docs/contracts/`. Both sides then build against the contract
   independently.
2. **Separate repositories = separate lanes.** Firmware, AI, mobile, and web
   rarely touch the same files.
3. **Backend is shared, so it is split by module.** NestJS modules (`devices`,
   `commands`, `telemetry`, `images`, `auth`, …) map to issues, and each issue
   has one assignee. Two people should not have open branches on the same module.
4. **Prisma schema is the hot spot.** Schema changes get their own small PR,
   merged quickly, so the other developer can rebase. Never edit a migration
   that has already been merged; add a new one.
5. **Simulators unblock each other.** A device simulator script (publishes fake
   heartbeats and acks) lets backend/app work proceed without hardware. A fake
   AI responder lets the pipeline be tested without a model.
6. **Board hygiene.** Move your card when status changes. Anything In Progress
   for more than 3 days gets split or discussed.

## Board columns

`Backlog → Todo → In Progress → Code Review → Testing → Done`

## Never commit

`.env` files, passwords, API keys, tokens, private keys, device certificates,
production credentials, or model weights over 50 MB (use releases/LFS/storage).
