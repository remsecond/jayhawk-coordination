# Hermes Recon — Shared Surface (Hostinger)

**Date:** 2026-05-07
**Source:** Hermes agent (running as `hermes-agent-bnz0-hermes-agent-1` on Hostinger), driven by Tab.
**Purpose:** Establish ground truth for `contracts/SHARED_SURFACE_HOSTINGER_V0.md` and `docs/HOSTINGER_DEPLOYMENT_NOTES.md`. Captured per the repo's operating rule: chat is not a source of truth.

This file is **verbatim** Hermes output and **summarized** consolidated findings. Where commands ran on Hermes, I record what Hermes reported; this isn't reproduced or rerun in this repo.

---

## Phase 1 — initial recon (Hermes vantage, ro mount only)

### (1) `ls -la /openclaw-source`

```
total 76
drwx------ 13 1000 1000 4096 May  1 05:16 .
drwxr-xr-x  1 root root 4096 May  5 01:25 ..
drwxr-xr-x  3 1000 1000 4096 May  1 01:17 agents
drwxr-xr-x  2 1000 1000 4096 May  1 01:19 canvas
drwx------  2 1000 1000 4096 May  1 06:16 credentials
drwxr-xr-x  2 1000 1000 4096 May  1 01:22 devices
drwxr-xr-x  2 1000 1000 4096 May  1 01:17 extensions
drwxr-xr-x  2 1000 1000 4096 May  1 01:20 identity
drwx------  2 1000 1000 4096 May  1 01:18 logs
-rw-------  1 1000 1000 3484 May  1 01:19 openclaw.json
-rw-------  1 1000 1000 3378 May  1 01:19 openclaw.json.bak
-rw-------  1 1000 1000 3143 May  1 01:19 openclaw.json.bak.1
-rw-------  1 1000 1000 2835 May  1 01:19 openclaw.json.bak.2
-rw-------  1 1000 1000 2694 May  1 01:18 openclaw.json.bak.3
-rw-------  1 1000 1000 3484 May  1 01:19 openclaw.json.last-good
drwxr-xr-x  3 1000 1000 4096 May  1 01:17 skills
drwx------  2 1000 1000 4096 May  1 01:19 tasks
drwx------  2 1000 1000 4096 May  1 06:16 telegram
drwxr-xr-x  5 1000 1000 4096 May  1 01:23 workspace
```

### (2) `ls -la /openclaw-source/credentials/`

```
total 12
drwx------  2 1000 1000 4096 May  1 06:16 .
drwx------ 13 1000 1000 4096 May  1 05:16 ..
-rw-------  1 1000 1000  348 May  1 06:16 telegram-pairing.json
```

### (3) Presence check for repo-layout dirs on the live surface

```
handoffs: absent
contracts: absent
docs: absent
extension-snapshot: absent
replit-sidecar-snapshot: absent
inbox: absent
outbox: absent
```

### (4) `/proc/mounts` for `openclaw-source` and `opt/data`

```
/dev/sda1 /openclaw-source ext4 ro,relatime,discard,errors=remount-ro,commit=30 0 0
/dev/sda1 /opt/data        ext4 rw,relatime,discard,errors=remount-ro,commit=30 0 0
```

### (5) `findmnt /openclaw-source`

```
TARGET           SOURCE                                          FSTYPE OPTIONS
/openclaw-source /dev/sda1[/docker/openclaw-iemy/data/.openclaw] ext4   ro,relatime,discard,errors=remount-ro,commit=30
```

### (6) Git on `/openclaw-source/workspace`

```
fatal: detected dubious ownership in repository at '/openclaw-source/workspace'
```
Same error for `git remote -v`, `git log -1 --oneline`, `git status -sb`. Not resolvable from Hermes without writing config (skipped per no-write constraint at recon time).

### (7) Filtered `mount` output

```
overlay on / type overlay (rw,relatime,lowerdir=...,upperdir=.../snapshots/51/fs,workdir=.../snapshots/51/work,nouserxattr)
/dev/sda1 on /openclaw-source type ext4 (ro,relatime,discard,errors=remount-ro,commit=30)
/dev/sda1 on /opt/data type ext4 (rw,relatime,discard,errors=remount-ro,commit=30)
/dev/sda1 on /etc/resolv.conf type ext4 (rw,relatime,discard,errors=remount-ro,commit=30)
/dev/sda1 on /etc/hostname    type ext4 (rw,relatime,discard,errors=remount-ro,commit=30)
/dev/sda1 on /etc/hosts       type ext4 (rw,relatime,discard,errors=remount-ro,commit=30)
```

### (8) `ls -la /openclaw-source/workspace/.openclaw/`

```
total 12
drwxr-xr-x 2 1000 1000 4096 May  1 01:23 .
drwxr-xr-x 5 1000 1000 4096 May  1 01:23 ..
-rw-r--r-- 1 1000 1000   70 May  1 01:23 workspace-state.json
```

### Active write activity (last 1h / 24h, Hermes vantage)

- Last hour: `agents/main/sessions/sessions.json`, `agents/main/sessions/<uuid>.jsonl`
- Last 24h: same two files. No other writes visible to Hermes's uid.

### Network checks (Hermes vantage)

- `ss -tunap` — not installed/permitted; no socket info.
- `getent hosts openclaw` — no record.
- `getent hosts openclaw-iemy` — no record.
- DNS resolver: Docker embedded at 127.0.0.11 with `search .` and `ndots:0`. Sibling container hostnames don't resolve.

---

## Phase 2 — host-side recon (uid 1000 vantage, full visibility)

Tab forwarded the consolidated Phase-2 briefing on 2026-05-07. Verbatim summary of findings, mapped to the eight original questions:

### Q1 — Transport / mount mechanism

Single Linux bind-mount declared in Hermes's `docker-compose.yml` at `/docker/hermes-agent-bnz0/docker-compose.yml`. Binds host directory `/docker/openclaw-iemy/data/.openclaw` (owned `ubuntu:ubuntu` uid 1000) into Hermes's container at `/openclaw-source` with `ro` enforced by the kernel. Filesystem ext4, options `ro,relatime,discard,errors=remount-ro,commit=30`. No sshfs, NFS, FUSE, rsync, cron, git pull/push, S3, or overlay-on-top.

### Q2 — Same mount or two views

Two views of the same inode tree on the same filesystem. OpenClaw mounts `/docker/openclaw-iemy/data → /data` rw. Hermes mounts `/docker/openclaw-iemy/data/.openclaw → /openclaw-source` ro. Same bytes, kernel enforces ro on Hermes view. Latency synchronous (page-cache flush, milliseconds). No replication.

### Q3 — Source of truth

Hostinger box is canonical. `workspace/.git` is freshly initialized (`master`), **no remotes, no commits**. Compose files are the only authoritative config:
- `/docker/openclaw-iemy/docker-compose.yml`
- `/docker/hermes-agent-bnz0/docker-compose.yml`

### Q4 — What does Hermes write back

Nothing on the shared surface (mount is ro, would EROFS). Hermes's only writable path is `/opt/data` (her HERMES_HOME), backed by `/docker/hermes-agent-bnz0/data` — separate volume, OpenClaw cannot see it. **No return path today.** Active surface writes confined to `agents/main/sessions/sessions.json` + per-session `<uuid>.jsonl`.

### Q5 — Other agents on the roadmap

Two containers on this box: `hermes-agent-bnz0-hermes-agent-1` (up 13h at recon) and `openclaw-iemy-openclaw-1` (up 6 days). Five docker networks (3 stock + `hermes-agent-bnz0_default` 172.19.0.0/16 + `openclaw-iemy_default` 172.18.0.0/16). Compose-default networks not joined; no shared network.

### Q6 — Actual layout vs draft contract

Host vantage (uid 1000) confirms 11 dirs in `/docker/openclaw-iemy/data/.openclaw/`: `agents/`, `canvas/`, `credentials/`, `devices/`, `extensions/`, `identity/`, `logs/`, `skills/`, `tasks/`, `telegram/`, `workspace/`. Plus 6 root files (`openclaw.json` + 4 `.bak` + `.last-good`). Total ~1.4 MB. Mode 0700 on `credentials/`, `logs/`, `tasks/`, `telegram/`, and `.openclaw` root itself — Hermes (different uid in earlier phase) sees ~1.05 MB of the 1.4 MB.

Dirs the v0 draft proposed (`handoffs/`, `contracts/`, `docs/`, `extension-snapshot/`, `replit-sidecar-snapshot/`, `inbox/`, `outbox/`) are **all absent** from the live surface. The contract treats them as to-be-created on `/coordination`, not on `/openclaw-source`.

### Q7 — `credentials/` inventory

One file: `telegram-pairing.json`, 348 bytes, mode `-rw-------`, uid 1000, mtime `2026-05-01 06:16`. No API keys, tokens, or binaries. The 0700 dir mode + ro mount happens to double-lock it today (Tab's term: "double lock by accident"); v0 makes the boundary structural via the allowlist in §3 of the contract.

### Q8 — Watchdog / HITL / latency

No watchdog. `crontab -l` empty for root. `systemctl list-timers --all` shows 17 stock Ubuntu timers, none touching the surface. `/etc/cron.{d,daily,hourly,weekly,monthly}/` contain only stock packages. Tab is the human-in-the-loop. Latency synchronous.

### Side findings (non-blocking but logged)

1. **7 zombie processes** in MOTD; "system restart required." Plan a maintenance window.
2. **`.env` divergence:** `/opt/data/.env` (live) vs `/docker/hermes-agent-bnz0/.env` (compose, holds dead key). Harmless until `docker compose down && up` — the dead key would re-inject. Reconcile before any container recreate.
3. **`bottleneck` npm lib** ships its own `.env` inside `node_modules/`. Library boilerplate, not config. Noise.
4. **Git "dubious ownership"** will persist for any non-uid-1000 traversal of the workspace. Resolution baked into deployment notes §5 (safe.directory entrypoint hook).

---

## Implications for the contract

These findings produced the v0 contract draft and deployment notes:

1. **Per-subdir allowlist on Hermes's compose** beats restructuring OpenClaw's data dir. Confirmed by Q1+Q6 — OpenClaw is unchanged, Hermes's compose is the single audit point.
2. **Hermes uid → 2000.** Today's recon shows Hermes can `ls` 0700 uid-1000 dirs, meaning she runs as root or uid 1000. v0 fixes this for defense-in-depth.
3. **`/coordination` rw bind volume** is the only available bidirectional channel given Q5 (no shared network) and Q4 (no return path today). Filesystem is the only currently-feasible transport.
4. **`workspace/.git` is not a real git source yet** (Q3). Contract still wires safe.directory for forward compatibility, but doesn't depend on the repo having content.
5. **No watchdog (Q8)** → contract notes Tab is HITL; v0.1 may add a heartbeat checker that posts to `inbox/`.

---

*End of recon capture. The Phase-1 commands ran from Hermes vantage; the Phase-2 briefing was forwarded by Tab and represents host-side findings I did not run myself. Both phases are recorded for audit.*
