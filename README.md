# S2SC — Server-to-Server Copy Controller

Safe, resumable server-to-server file copy orchestration using rsync over SSH — with a web UI, job queue, dry-run safety gates, and conflict management. No agent software needed on endpoints.

> **Private source — this repository is a project showcase.**

---

## The Problem It Solves

Copying large datasets between servers usually means routing data through the machine running the command — slow, wasteful, and brittle on flaky connections. S2SC fixes this: the controller SSHes into the **source** host and triggers rsync there, so data flows **directly** from source to destination without touching the controller machine.

The other problem is safety. Bulk copies are hard to undo. S2SC enforces a dry-run gate before every transfer, requires explicit approval to start, and never deletes or overwrites by default.

---

## How It Works

```
Controller (Linux)
  │
  ├─── SSH ──► Source host
  │              │
  │              └─── rsync over SSH ──► Destination host
  │
  └─── Web UI ◄── Browser (any machine on the network)
```

1. **Create a job** — pick source endpoint, destination endpoint, paths
2. **Dry-run prep** — controller scans files, counts size, detects conflicts
3. **Review** — inspect totals and any existing-file conflicts in the UI
4. **Approve & start** — one click triggers the actual rsync
5. **Monitor** — live progress, transfer rate, and ETA in the UI
6. **Optional verify** — post-copy checksum dry-run confirms integrity

---

## Features

### Safety Model
- **Dry-run gate** — every job runs a prep scan before copy; copy never starts automatically
- **No deletes, no overwrites** — `--ignore-existing` by default; destination files are never touched
- **Destination must exist** — no accidental directory creation
- **Allowlist-only paths** — each endpoint declares its allowed base directories; paths outside are rejected
- **Conflict policy** — choose Ask / Skip / Stop for existing destination files; Ask requires an explicit decision before copy starts
- **Verify-only mode** — run a checksum dry-run against an already-copied destination; reports mismatches without writing anything

### Job Queue
- Sequential execution — one active transfer at a time
- Reorderable queue for waiting jobs
- Cancel, pause, and restart support
- Job states: `queued → preparing → ready → copying → finished / failed / canceled`

### Web UI
- Queue overview with live status, progress bar, rate, and ETA
- Job detail view with per-job rsync log and file list
- New job form — endpoint picker, path entry, policy and flag configuration
- Log viewer — per-job transfer log and file list log

### Admin UI
- Token management — generate and rotate API auth tokens
- Endpoint management — add/remove servers and their allowed base directories
- Admin password rotation without editing config files
- Session-based login with configurable TTL

### Transfer Options (per job)
| Option | Default |
|---|---|
| On existing files | Ask |
| Checksum verify | Off |
| Preserve permissions + timestamps | On |
| Write-in-place | Off |
| Auto-start after prep | Off |
| Verify after copy | Off |
| Bandwidth limit | None |
| Debug logging | Off |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.10+ / FastAPI |
| Web server | Uvicorn (ASGI) |
| Templating | Jinja2 |
| Database | SQLite (job state, logs) |
| Transfer engine | rsync over SSH (triggered remotely via Paramiko) |
| Auth | Token-based (queue UI) + session-based (admin UI) |
| Password storage | PBKDF2-SHA256 |
| Tests | pytest |
| Frontend | Vanilla JS + CSS (no framework dependency) |

---

## Architecture

```
app/
  main.py           FastAPI routes — queue UI, job API, admin UI
  config.py         Config loader + endpoint validation
  transfer.py       rsync argument builder + SSH execution via Paramiko
  queue.py          Sequential job queue, state machine
  state.py          In-memory job state + SQLite persistence
  storage.py        SQLite schema, reads/writes
  ssh.py            SSH connection pool + remote command runner
  security.py       Token auth, password hashing, session management
  validators.py     Path allowlist enforcement
  logging_setup.py  Per-job + global log routing

  templates/        Jinja2 HTML templates (queue, job detail, new job, admin, logs)
  static/           app.js + app.css (vanilla, no build step)

  tests/            pytest suite — transfer args, state transitions,
                    validators, admin password flow
```

**No agent on endpoints** — the controller SSHes in and runs a single rsync command. Source and destination only need rsync + SSH; no custom software installed.

**Config-driven endpoints** — servers are declared in `config.json` with host, user, port, OS type, and allowed base directories. No code changes to add a new server.

**SQLite for persistence** — job state, logs, and file lists stored locally. No external database required.

---

## Key Engineering Decisions

**Why trigger rsync on the source instead of the controller?**
Routing large transfers through the controller wastes bandwidth and adds a bottleneck. Running rsync on the source means data moves directly between endpoints — the controller only needs SSH access to the source, not to both.

**Why a dry-run gate before every copy?**
Bulk file operations are hard to reverse. Requiring a prep scan and explicit approval before any data moves prevents accidents on production storage. You always see exactly what will transfer before it happens.

**Why no deletes or overwrites by default?**
S2SC is a copy tool, not a sync tool. The default behavior is additive-only. Destination files are never touched unless the operator explicitly changes the conflict policy — and even then, the policy applies per-job, not globally.

**Why SQLite instead of a heavier database?**
S2SC is a controller meant to run on a single Linux machine. SQLite is zero-infrastructure — no service to manage, no connection pool, no separate install. Job state and logs persist across restarts without any setup.

**Why vanilla JS on the frontend?**
The UI is straightforward: a queue list, a job detail view, and a form. A framework would add a build step and bundle complexity for no benefit. Plain JS keeps the frontend deployable without Node.js on the controller.

---

## Security Notes

- `config.json` is excluded from version control — `config.example.json` is provided as a template
- Auth tokens are required for all queue and job API endpoints
- Admin UI uses separate session-based login with PBKDF2-SHA256 hashed passwords
- Endpoint path allowlists are enforced server-side — clients cannot reference arbitrary paths
- SSH uses key-based authentication; no passwords over the wire

---

## Status

Active development. Private source — this repository is a project showcase.
