# slate-server

The self-hosted server for [Slate](../slate-app), the time-block planner. You run it with Docker on
your own machine or server, and it keeps the data of several users under your control.

> **Status: planning.** This repository currently holds only the documentation. No code exists yet.
> Start with [docs/server-plan.md](docs/server-plan.md), which describes the architecture, the
> sync protocol, the API and the milestones.

## What it will do

- Accounts for several users (invite only by default), signed in with email and password.
- Sync of blocks, shifts, rotations and settings with the Slate app, when the app is set to
  **Your own Slate server**. Phone-calendar and Google events are never sent here.
- Backup and restore in the same file format the app exports.
- One `docker compose up` to run the API, Postgres and the TLS proxy.

## What it will not do

- It does not call Google or hold Google tokens. Google sync stays on the user's device.
- It does not track users or load third-party scripts.

## Planned quick start

This is the target for milestone M0. It does not work yet.

```bash
cp .env.example .env          # domain, database password, SMTP (later)
docker compose up -d
docker compose exec api slate-server admin create you@example.com
```

## Repository layout (planned)

| Path | Contents |
|---|---|
| `docs/` | Plans, API reference, operations guide |
| `api/` | The HTTP API (Node 22, TypeScript, Fastify) |
| `compose.yaml` | Postgres 16, the API, Caddy for TLS |
| `packages/core/` | Recurrence and validation shared with the Slate app |

Only `docs/` and this README exist so far.

## Development

Nothing to run yet. The first milestone is M0 in [docs/server-plan.md](docs/server-plan.md#9-milestones).

## Licence

To be decided before the first public release.
