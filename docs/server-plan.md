# Slate server: a self-hosted, multi-user backend

Goal: a Slate server that you run yourself with Docker, on your own machine or VPS, so that the
data of several users lives under your control. It also becomes the second way to sign in to the
apps ("Sign in with Slate", in progress in Settings › Account). Google sign-in stays as it is:
tokens remain on the device, and the server never holds Google tokens.

Status: **plan only**. No server code exists yet. The repository would be a new project,
`slate-server`, next to the other projects in `~/dev`.

## 1. Principles

1. **You own the data.** One Docker Compose file starts everything. Backups are plain files you
   copy off the machine.
2. **Offline first.** The apps keep working without the server, as they do now. The server is a
   sync target, not the place where the app reads from.
3. **No Google tokens on the server.** The server does not call Google. Calendar sync stays
   device-side, so the server cannot act on a user's behalf at Google.
4. **Users cannot see each other's data.** Every query is scoped to the signed-in user. The
   database also enforces it (row-level security), so a bug in one query cannot leak rows.
5. **Small and boring.** Postgres, one API process, one reverse proxy. No queue, no cache, no
   third-party tracker.

## 2. Architecture

```
 phone (Slate app) ─┐                       ┌─ Postgres 16 (volume: db)
                    ├─ HTTPS ─ Caddy ─ API ─┤
 browser (website) ─┘   (TLS, certs)  (Node) └─ backups (pg_dump → volume/backups)
```

- **Caddy** terminates TLS (automatic certificates for your domain) and forwards to the API.
- **API** is Node 22 + TypeScript + Fastify. It shares the recurrence and mapping code with the
  apps through a small package (`@slate/core`), so the rules are written once.
- **Postgres 16** with row-level security enabled on every table.
- **Docker Compose** runs the three services. Images are built from the repository; the
  Postgres and Caddy images are pinned by tag.

Why these: the apps are TypeScript, so one language and shared logic is less work than a second
stack. Postgres is what the website already uses (Supabase is Postgres), which makes a later
migration of the website's data straightforward.

## 3. Accounts and sign-in

- **Email and password.** Passwords hashed with Argon2id. Registration is **invite only** by
  default: you create users from a command line (`slate-server user add`) or issue invite codes.
- **Sessions.** The website uses an `httpOnly`, `Secure`, `SameSite=Lax` cookie. The app stores a
  refresh token in the device's secure preferences and sends a short-lived access token with each
  request. Logging out revokes the session on the server.
- **Rate limits** on login and registration, per IP and per account.
- **Password reset** by email needs an SMTP account (see decisions).
- **Later:** passkeys and two-factor. Not in the first milestone.
- The **Google sign-in** is unrelated to this and stays on the device.

## 4. Data model

Mirrors the app's storage, so a backup file from the app imports directly.

| Table | Purpose | Key columns |
|---|---|---|
| `users` | Accounts | `id`, `email` (unique), `password_hash`, `is_admin`, `created_at`, `deleted_at` |
| `devices` | One row per phone or browser that synced | `id`, `user_id`, `name`, `last_seen_at` |
| `sessions` | Refresh tokens | `id`, `user_id`, `device_id`, `token_hash`, `expires_at`, `revoked_at` |
| `blocks` | Events and series (as in the web app) | `id`, `user_id`, title, category, colour, day, time, duration, `week_offset`, notes, recurrence fields, `skipped_weeks` (array), `version`, `updated_at`, `deleted_at` |
| `shifts` | Shift definitions | `id`, `user_id`, title, start/end, `break_minutes`, colour, `version`, `updated_at`, `deleted_at` |
| `shift_assignments` | A shift placed on a day | `id`, `user_id`, `shift_id`, `shift_date`, `version`, `updated_at`, `deleted_at` |
| `rotations` | Cycles | `id`, `user_id`, title, `pattern` (jsonb), `version`, `updated_at`, `deleted_at` |
| `settings` | One row per user | `user_id`, `document` (jsonb: theme, pay, language, view), `version` |
| `changes` | Change log for sync | `seq` (bigserial), `user_id`, `entity`, `entity_id`, `op` |

- `version` is incremented by the server on every write. `deleted_at` keeps tombstones so
  deletions sync too. Tombstones are purged after 90 days once every device has pulled past them.
- Every table has `user_id` and a Postgres policy `user_id = current_setting('app.user_id')::uuid`.
  The API sets that setting per transaction after it verifies the session.
- Calendar-device ids (`calendar.targetCalendarId` and the Google push state) never leave the
  device, as now.

## 5. Sync protocol

Pull-then-push, cursor based, offline tolerant.

- **Pull:** `GET /v1/sync/changes?since=<seq>` returns every change after `seq` for the user, in
  order, with the current row. The client applies them and stores the new cursor.
- **Push:** `POST /v1/sync/mutations` with a batch of local mutations. Each has a client-generated
  id (UUID), the entity, the `base_version` it was edited from, and the new fields. The server
  applies each one in a transaction:
  - if `base_version` equals the server's version: apply, bump `version`, log a change;
  - if not: **last writer wins per record** by server arrival order, the client gets the current
    row back and marks its local copy as superseded. Field-level merge is a later improvement.
- **Idempotency:** a mutation id that was already applied returns the stored result, so retries
  after a lost response are safe.
- **Ordering:** `seq` is global per user, so a pull never misses a change.
- **Series:** a recurring series is one row, as in the app. Skipped occurrences are a list on the
  row, so an edit of one occurrence is one row update.
- **Full sync** (first login, or a new device): pull from `seq=0`. The app then pushes its own
  local rows that the server does not know (by id), so nothing is lost on first sign-in.

## 6. API surface (v1)

| Method | Path | Notes |
|---|---|---|
| POST | `/v1/auth/login` | Returns access token, sets refresh cookie or returns refresh token (app) |
| POST | `/v1/auth/refresh` | Rotates the refresh token |
| POST | `/v1/auth/logout` | Revokes the session |
| GET | `/v1/me` | Profile and devices |
| GET | `/v1/sync/changes?since=` | Pull |
| POST | `/v1/sync/mutations` | Push |
| GET | `/v1/backup` | Full export in the app's backup format |
| POST | `/v1/backup` | Import (validated like the app does) |
| GET | `/healthz` | For Docker health checks and uptime monitors |

All responses are JSON. Errors use a stable `{ "error": { "code", "message" } }` shape.

## 7. How the apps connect

- **Android/iOS app:** Settings › Account › **Sign in with Slate** asks for the server address
  (default: your domain), email and password. After login the app syncs on resume and after each
  change (debounced), like the Google push. Offline changes queue locally.
- **Website:** a second sign-in provider next to Supabase, or a replacement for it later. Decided
  when the server is running.
- **Google** keeps working side by side. Its data never goes through the Slate server.

## 8. Operations

- **Install:** `docker compose up -d` with an `.env` file (domain, database password, SMTP).
  Caddy obtains the certificate on first start.
- **Create the first admin:** `docker compose exec api slate-server admin create you@example.com`.
- **Backups:** nightly `pg_dump` into a volume, kept 14 days, plus a weekly copy you move off the
  machine. A restore is documented and tested once.
- **Updates:** pull the new image tag, run migrations (`slate-server migrate`), restart. Migrations
  are forward-only and checked in CI.
- **Logs:** JSON to stdout; no request bodies, no passwords, no tokens. Access logs keep the IP
  for 30 days.
- **Monitoring:** `/healthz` from an uptime checker; Docker health checks restart a failed API.
- **Privacy:** no analytics, no third-party scripts. Users can export everything and delete their
  account (which deletes their rows and tombstones at once).

## 9. Milestones

| # | Milestone | Done when |
|---|---|---|
| M0 | Repo, compose file, Postgres with RLS, health check | `docker compose up` gives a healthy `/healthz` on a local domain |
| M1 | Accounts: invite, login, refresh, logout, admin CLI | A test suite covers login, wrong password, revoked session |
| M2 | Sync API: changes, mutations, idempotency, last-writer-wins | Tests: two devices converge; retried mutation is applied once; cross-user access is refused |
| M3 | Backup export and import in the app's format | Round-trip test with a file from the app |
| M4 | App client: Sign in with Slate, sync engine | Two phones converge; offline edits sync later |
| M5 | Website client | Same behaviour on the web |
| M6 | Passkeys, 2FA, field-level merge | Optional |

Each milestone ends with the tests passing in CI and a deploy to a test domain.

## 10. Decisions needed from you

1. **Domain and host.** Which domain (e.g. `slate.example.com`) and which machine (home server,
   VPS)? A VPS with a static IP makes certificates and uptime simpler.
2. **Registration.** Invite only (recommended), or open?
3. **Email for password reset.** Which SMTP provider? Without it, a reset is done by an admin.
4. **Website.** Keep Supabase for the website and add the server as a second sign-in, or move the
   website to the server later?
5. **Languages and names.** Keep the current six languages and Italian copy for the server's own
   admin messages?
6. **Shared code package.** OK to create `@slate/core` (recurrence, mapping, validation) shared by
   the app, the website and the server? It changes how `slate-app` and `slate` import those files.

## 11. What I would build first

M0 and M1 in a new `slate-server` repository, with the same validation rules the app uses, so the
first client work (M4) starts from tested endpoints.

## 12. The two ways to use Slate (decided)

1. **On this phone (local).** Everything is stored on the device. Optional: the phone's calendars
   (read, and edit or delete events there) and Google (read in real time; copy Slate items to a
   "Slate" calendar). Nothing from either calendar is sent to a server.
2. **Your own Slate server (hosted, in progress).** Blocks and shifts created in Slate sync with
   the server you configure. Phone-calendar events are never synced with it: they are shown only on
   the device that has access to them. Google events are not synced either.

Tags: an event shows **Device** or **Google** when it comes from there. An event with no tag was
created in Slate.
