# Hostinger Deployment Notes — Shared Coordination Surface v0

> Companion to `contracts/SHARED_SURFACE_HOSTINGER_V0.md`. This file is the **executable migration plan** for the Hostinger box currently running `openclaw-iemy` and `hermes-agent-bnz0`. Hand the steps below to whoever is wiring it up.

## 0. Pre-flight — verified facts (Phase-2 recon, 2026-05-07)

These are observed truths the plan depends on. Re-verify if anything has been changed since the recon date.

- Host: single Hostinger box. Two containers: `openclaw-iemy-openclaw-1` and `hermes-agent-bnz0-hermes-agent-1`.
- Compose files:
  - `/docker/openclaw-iemy/docker-compose.yml`
  - `/docker/hermes-agent-bnz0/docker-compose.yml`
- Today's shared surface: single bind mount of `/docker/openclaw-iemy/data/.openclaw → /openclaw-source` ro, declared in **Hermes's** compose.
- OpenClaw data root: `/docker/openclaw-iemy/data/.openclaw/`, owned `ubuntu:ubuntu` (uid 1000).
- Hermes writable home: `/opt/data` in container ↔ `/docker/hermes-agent-bnz0/data` on host.
- Networks: `openclaw-iemy_default` (172.18.0.0/16) and `hermes-agent-bnz0_default` (172.19.0.0/16). Not joined. No DNS between containers.
- No cron, no systemd timers touching the surface. No watchdog.
- `workspace/.git`: fresh repo, no remotes, no commits. Not a real source of truth today.
- Side findings to clean up while you're in there (unrelated to surface, but in the way):
  - `.env` divergence: `/opt/data/.env` (live) vs `/docker/hermes-agent-bnz0/.env` (stale, holds dead key). Reconcile before any `docker compose down && up` or the dead key gets re-injected.
  - 7 zombie processes; "system restart required" in MOTD. Plan a maintenance window.

## 1. Pre-change backups

Before touching anything:

```sh
# Backup OpenClaw's full data dir (read-only snapshot is fine)
sudo tar -czf /root/backup-openclaw-data-$(date -u +%Y%m%dT%H%M%SZ).tar.gz \
  -C /docker/openclaw-iemy data

# Backup both compose files
sudo cp /docker/openclaw-iemy/docker-compose.yml \
        /root/backup-openclaw-compose-$(date -u +%Y%m%dT%H%M%SZ).yml
sudo cp /docker/hermes-agent-bnz0/docker-compose.yml \
        /root/backup-hermes-compose-$(date -u +%Y%m%dT%H%M%SZ).yml

# Capture current container state for the audit trail
docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}\t{{.RunningFor}}' | sudo tee /root/backup-container-state-$(date -u +%Y%m%dT%H%M%SZ).txt
```

## 2. Create the shared coordination volume

Two-way coordination needs a volume **both** containers can write to. We'll use a host bind path so it's visible/auditable from the host.

```sh
# Create the coordination dir + structure on the host
sudo mkdir -p /docker/coordination-shared/{from-claw,from-hermes,shared,inbox,contracts}

# Group both container uids can join (Claw=1000, Hermes=2000 — see §4)
sudo groupadd -g 3000 coordination 2>/dev/null || true

# Ownership: subdir owners enforce the writer rule
sudo chown -R root:coordination /docker/coordination-shared
sudo chown 1000:coordination    /docker/coordination-shared/from-claw
sudo chown 2000:coordination    /docker/coordination-shared/from-hermes
sudo chmod 0775 /docker/coordination-shared
sudo chmod 0775 /docker/coordination-shared/{shared,inbox}
sudo chmod 0755 /docker/coordination-shared/{from-claw,from-hermes,contracts}

# Seed contracts/ from this repo on first run (re-run any time the repo updates)
# Replace REPO_URL with the canonical jayhawk-coordination remote
sudo git -C /tmp clone --depth 1 REPO_URL jayhawk-coordination-snap || true
sudo cp -r /tmp/jayhawk-coordination-snap/contracts/* /docker/coordination-shared/contracts/
sudo git -C /tmp/jayhawk-coordination-snap rev-parse HEAD | sudo tee /docker/coordination-shared/contracts/.commit
sudo rm -rf /tmp/jayhawk-coordination-snap
```

## 3. Update OpenClaw's compose (mount coordination volume rw)

Edit `/docker/openclaw-iemy/docker-compose.yml`. **Add** the coordination volume to the OpenClaw service. Do not change anything else about how OpenClaw runs.

```yaml
services:
  openclaw:
    # ... existing keys preserved ...
    volumes:
      # ... existing volumes preserved ...
      - /docker/coordination-shared:/coordination:rw
    # If the service does not already declare a uid, add:
    user: "1000:3000"   # uid 1000 (existing), gid 3000 (coordination)
```

**Do not** add the coordination group to OpenClaw via `group_add` if `user:` already specifies a primary gid you want to keep — instead use `group_add: ["3000"]` and leave `user:` alone:

```yaml
    user: "1000:1000"
    group_add:
      - "3000"
```

Pick whichever form matches the existing service definition. The goal: OpenClaw's process is uid 1000 with supplementary gid 3000.

## 4. Update Hermes's compose (allowlist mounts + new uid + coordination volume)

Edit `/docker/hermes-agent-bnz0/docker-compose.yml`. This is the load-bearing change. Three things happen here:

**a) Replace the single `/openclaw-source` ro mount** with a per-subdir allowlist (matches `contracts/SHARED_SURFACE_HOSTINGER_V0.md` §3):

```yaml
services:
  hermes-agent:
    # ... existing keys preserved EXCEPT the volumes block ...
    user: "2000:3000"           # was: root or uid 1000 — change to dedicated uid 2000
    group_add:
      - "3000"                  # coordination group
    volumes:
      # Hermes's own writable home — UNCHANGED
      - /docker/hermes-agent-bnz0/data:/opt/data:rw

      # NEW: bidirectional coordination surface
      - /docker/coordination-shared:/coordination:rw

      # REPLACED: per-subdir ro allowlist of OpenClaw's public set.
      # The old single mount of `.openclaw → /openclaw-source` is REMOVED.
      - /docker/openclaw-iemy/data/.openclaw/agents:/openclaw-source/agents:ro
      - /docker/openclaw-iemy/data/.openclaw/canvas:/openclaw-source/canvas:ro
      - /docker/openclaw-iemy/data/.openclaw/devices:/openclaw-source/devices:ro
      - /docker/openclaw-iemy/data/.openclaw/extensions:/openclaw-source/extensions:ro
      - /docker/openclaw-iemy/data/.openclaw/skills:/openclaw-source/skills:ro
      - /docker/openclaw-iemy/data/.openclaw/tasks:/openclaw-source/tasks:ro
      - /docker/openclaw-iemy/data/.openclaw/workspace:/openclaw-source/workspace:ro
      - /docker/openclaw-iemy/data/.openclaw/openclaw.json:/openclaw-source/openclaw.json:ro
```

**Crucial:** there is NO mount for `credentials/`, `telegram/`, `identity/`, or `logs/`. Their absence from the allowlist is the security boundary. Do not add them.

**b) Hermes's filesystem in `/opt/data` must be readable/writable by uid 2000** after the uid change. Fix ownership before bringing the container up:

```sh
sudo chown -R 2000:2000 /docker/hermes-agent-bnz0/data
```

**c) Reconcile the .env divergence flagged in §0** before `docker compose up`:

```sh
# Make the compose-level .env match the live one (or pick the canonical source)
sudo diff /opt/... /docker/hermes-agent-bnz0/.env   # inspect first
# Then copy the live values from /docker/hermes-agent-bnz0/data/.env (live)
# to /docker/hermes-agent-bnz0/.env (compose) — or the other direction, your call.
```

## 5. Add the git safe.directory entrypoint hook (Hermes)

In Hermes's container image (or via a small init step in the compose `command:` / `entrypoint:`), ensure the agent runs the following on first start:

```sh
git config --global --add safe.directory '/openclaw-source/workspace'
git config --global --add safe.directory '/coordination/*'
```

If you can't bake it into the image, the simplest path is a one-shot `command:` wrapper:

```yaml
    entrypoint:
      - /bin/sh
      - -c
      - |
        git config --global --add safe.directory '/openclaw-source/workspace'
        git config --global --add safe.directory '/coordination/*'
        exec <original-entrypoint-here>
```

Replace `<original-entrypoint-here>` with whatever Hermes currently runs. If you don't know it, `docker inspect hermes-agent-bnz0-hermes-agent-1 --format '{{.Config.Entrypoint}} {{.Config.Cmd}}'` will tell you before you take it down.

## 6. Bring it up

```sh
# Stop both stacks cleanly
sudo docker compose -f /docker/hermes-agent-bnz0/docker-compose.yml down
sudo docker compose -f /docker/openclaw-iemy/docker-compose.yml down

# Bring OpenClaw up first (Hermes depends on its files existing)
sudo docker compose -f /docker/openclaw-iemy/docker-compose.yml up -d

# Verify OpenClaw is healthy before continuing
sudo docker ps --filter name=openclaw

# Then Hermes
sudo docker compose -f /docker/hermes-agent-bnz0/docker-compose.yml up -d
sudo docker ps --filter name=hermes
```

## 7. Verification — run these from inside Hermes after startup

```sh
# 1. Allowlist worked: should see exactly 7 dirs + openclaw.json under /openclaw-source.
docker exec hermes-agent-bnz0-hermes-agent-1 ls -la /openclaw-source

# Expected:
#   agents canvas devices extensions skills tasks workspace openclaw.json
# Expected ABSENT: credentials telegram identity logs

# 2. Credentials are gone from Hermes's view.
docker exec hermes-agent-bnz0-hermes-agent-1 ls /openclaw-source/credentials 2>&1
# Expected: "No such file or directory"

docker exec hermes-agent-bnz0-hermes-agent-1 ls /openclaw-source/telegram 2>&1
# Expected: "No such file or directory"

# 3. Hermes is uid 2000.
docker exec hermes-agent-bnz0-hermes-agent-1 id
# Expected: uid=2000 ... gid=2000 ... groups=2000,3000(coordination)

# 4. Coordination volume is rw from Hermes side.
docker exec hermes-agent-bnz0-hermes-agent-1 sh -c \
  'echo "hermes-write-test $(date -u)" > /coordination/from-hermes/_writetest && cat /coordination/from-hermes/_writetest && rm /coordination/from-hermes/_writetest'

# 5. Coordination volume is rw from Claw side.
docker exec openclaw-iemy-openclaw-1 sh -c \
  'echo "claw-write-test $(date -u)" > /coordination/from-claw/_writetest && cat /coordination/from-claw/_writetest && rm /coordination/from-claw/_writetest'

# 6. Cross-visibility: Claw can read what Hermes wrote.
docker exec hermes-agent-bnz0-hermes-agent-1 sh -c 'echo "from-hermes-msg" > /coordination/from-hermes/handshake.txt'
docker exec openclaw-iemy-openclaw-1 cat /coordination/from-hermes/handshake.txt
# Expected: "from-hermes-msg"

# 7. Hermes cannot WRITE to Claw's tree (correct).
docker exec hermes-agent-bnz0-hermes-agent-1 sh -c 'echo x > /coordination/from-claw/_should-fail' 2>&1
# Expected: Permission denied

# 8. Hermes cannot WRITE to /openclaw-source (correct, ro mount).
docker exec hermes-agent-bnz0-hermes-agent-1 sh -c 'echo x > /openclaw-source/openclaw.json' 2>&1
# Expected: Read-only file system

# 9. Git safe.directory works.
docker exec hermes-agent-bnz0-hermes-agent-1 git -C /openclaw-source/workspace status 2>&1
# Expected: status output (or "no commits yet"), NOT "dubious ownership"
```

If all nine pass, the surface is live and meets contract.

## 8. Rollback

If anything goes wrong, reverting is two file restores + a recreate:

```sh
sudo cp /root/backup-hermes-compose-<TS>.yml   /docker/hermes-agent-bnz0/docker-compose.yml
sudo cp /root/backup-openclaw-compose-<TS>.yml /docker/openclaw-iemy/docker-compose.yml
sudo docker compose -f /docker/hermes-agent-bnz0/docker-compose.yml down
sudo docker compose -f /docker/openclaw-iemy/docker-compose.yml down
sudo docker compose -f /docker/openclaw-iemy/docker-compose.yml up -d
sudo docker compose -f /docker/hermes-agent-bnz0/docker-compose.yml up -d
```

Data on `/docker/coordination-shared/` is untouched by rollback; leave it for the next attempt or `rm -rf` if abandoning.

## 9. Post-deploy housekeeping

These aren't blockers but should be on the next-pass list:

- Plan a host reboot for the "system restart required" + 7 zombies.
- Decide whether `inbox/` becomes per-agent (open question §11.2 of the contract).
- Define heartbeat cadence (open question §11.3).
- Decide whether to add a fs-watcher sidecar so Claw reacts to `from-hermes/` writes promptly (open question §11.7).

## 10. What this does NOT do

- Does not change OpenClaw's container or its data layout.
- Does not join the two compose networks. Coordination remains filesystem-only by design.
- Does not introduce any cron, timer, or sync job. Latency is synchronous (kernel page-cache flush).
- Does not push or pull anything from a remote git host. The jayhawk-coordination repo is the contract source; the live `/coordination/contracts/` is a snapshot pinned by `.commit`.

---

*End of deployment notes. Pair with `contracts/SHARED_SURFACE_HOSTINGER_V0.md` for the why; this file is the how.*
