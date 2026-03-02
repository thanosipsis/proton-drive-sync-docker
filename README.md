# Proton Drive Sync Docker Stack

Docker stack for running `proton-drive-sync` against a local Kopia repository.

This wrapper keeps Proton credentials and sync state in local persistent volumes, and builds from the local `./source` checkout.

## Prerequisites

- Docker Engine + Docker Compose plugin
- A Proton account
- A local Kopia repository on the host

## Install (Docker)

1. Go to this stack directory:
   - `cd /home/thano/srv/docker/proton-drive-sync`
2. Create `.env` from the template:
   - `cp .env.example .env`
3. Set `KEYRING_PASSWORD` in `.env`:
   - `openssl rand -base64 32`
4. Confirm `SYNC_DIR_KOPIA` in `.env` points to your host Kopia repository path.
5. Build image:
   - `docker compose build proton-drive-sync`
6. Authenticate once:
   - `docker compose run --rm --entrypoint proton-drive-sync proton-drive-sync auth`
7. Start service:
   - `docker compose up -d proton-drive-sync`
8. Open dashboard:
   - `http://<server-ip>:4242`

## Initial Dashboard Setup

Add one sync mapping:

- Local path: `/data/kopia-repository`
- Remote root: your Proton destination (example: `/Backups/Kopia`)

## Changes In This Fork (vs upstream main)

- Docker-first deployment wrapper with compose/env and persistent state volumes.
- Local source image build (`./source`) instead of relying on external prebuilt image availability.
- Job execution timeout to prevent hung worker operations.
- Recovery of stale in-memory active task slots after timeout.
- Periodic blocked-job recovery pass in the sync loop.
- Auto-requeue for blocked jobs caused by `Invalid access token` (with cooldown).
- Idempotent handling for create-file conflicts:
  - `A file or folder with that name already exists` is treated as synced instead of endlessly retried.
- Run-lock hardening for container restart PID reuse:
  - stale `running_pid` self-lock is auto-cleared.
- Docker runtime hardening in compose (`no-new-privileges`, `pids_limit`, controlled stop grace period).

## Authentication Notes

During `auth`, you may see:

- `Failed to fork child session: Cannot fork child session`
- `Falling back to parent session: ...`

This can be non-fatal if startup logs later show successful auth.

Check:

- `docker compose logs --tail=80 proton-drive-sync`
- Look for: `Authenticated as <user>`

If startup still fails with `Invalid access token`:

1. Re-run auth:
   - `docker compose run --rm --entrypoint proton-drive-sync proton-drive-sync auth`
2. Start again:
   - `docker compose up -d proton-drive-sync`

## Operations

- Status:
  - `docker compose ps`
  - `curl -s http://127.0.0.1:4242/api/stats`
- Logs:
  - `docker compose logs -f proton-drive-sync`
- Restart:
  - `docker compose restart proton-drive-sync`
- Rebuild + restart:
  - `docker compose up -d --build proton-drive-sync`
- Stop:
  - `docker compose stop proton-drive-sync`

## Troubleshooting

If queue health is bad (`blocked > 0`, `processing` stuck, or retries never drain):

1. Rebuild and restart:
   - `docker compose up -d --build proton-drive-sync`
2. Check stats:
   - `curl -s http://127.0.0.1:4242/api/stats`
3. Check logs for auth or conflict errors:
   - `docker compose logs --tail=200 proton-drive-sync`

## Security

- Do not commit `.env` or `data/` directories.
- `data/config` contains encrypted credentials and config.
- `data/state` contains sync state and internal SQLite metadata.
