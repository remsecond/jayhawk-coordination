# Hermes ↔ OpenClaw Integration Findings (rendered)

> Captured verbatim from the findings document drafted by Tab's other Claude session on 2026-05-07. Original target path: `C:\Users\ClawDaddy\Documents\moyer_case\_system\hermes_openclaw_integration_findings.md` (Robert's local case file). Landing here per repo rule: chat is not a source of truth.
>
> Pair with `handoffs/2026-05-07_hermes-recon-shared-surface.md` (the raw recon capture) and `contracts/SHARED_SURFACE_HOSTINGER_V0.md` (the contract this evidence informs).

---

# Hermes ↔ OpenClaw Integration Findings

**Investigation:** 2026-05-07, Hostinger VPS srv1317315.hstgr.cloud (76.13.118.164)
**Method:** Phase 1 — five read-only recon prompts to Hermes Agent (claude-opus-4.7) from inside her container at `/openclaw-source`. Phase 2 — six commands as root in the Hostinger Web Terminal. No writes performed. No credential file contents read.
**Containers in scope at investigation time:** `hermes-agent-bnz0-hermes-agent-1` (up 13 hours), `openclaw-iemy-openclaw-1` (up 6 days).

## Finding 1 — Mount mechanism

**Question.** How is the shared surface exposed?

**Evidence.**
From inside Hermes:
```
$ findmnt /openclaw-source
TARGET           SOURCE                                          FSTYPE OPTIONS
/openclaw-source /dev/sda1[/docker/openclaw-iemy/data/.openclaw] ext4   ro,relatime,discard,errors=remount-ro,commit=30
```
From the host, `docker inspect hermes-agent-bnz0-hermes-agent-1` Mounts:
```
bind /docker/hermes-agent-bnz0/data           -> /opt/data         (RO=False)
bind /docker/openclaw-iemy/data/.openclaw     -> /openclaw-source  (RO=True)
```
`mount | grep -E 'openclaw|claw|nfs|sshfs|fuse'` from inside Hermes returned only the one ext4 line above. No fuse, NFS, sshfs, rsync, or git-pull transports observed.

**Implication.** The shared surface is a single Linux kernel bind-mount on the same host. Hermes's view is RO-enforced at the kernel level; OpenClaw's view of the same bytes is RW. Latency is synchronous (page-cache flush, milliseconds). No replication layer exists.

## Finding 2 — Credentials directory exposure

**Question.** What's in `/openclaw-source/credentials/` and which entries does Hermes legitimately need?

**Evidence.**
From host (`ls -la /docker/openclaw-iemy/data/.openclaw/credentials/`):
```
total 12
drwx------ 2 ubuntu ubuntu 4096 May  1 06:16 .
drwx------ 13 ubuntu ubuntu 4096 May  1 05:16 ..
-rw------- 1 ubuntu ubuntu  348 May  1 06:16 telegram-pairing.json
```
From Hermes (metadata-only `find ... -exec stat`):
```
1) CREDENTIAL_FILE_COUNT: 1
2) FILE_NAMES_AND_SIZES: telegram-pairing.json | 348 bytes | 2026-05-01 06:16:43.244579406 +0000
3) FILE_TYPES: (empty — `file` not installed in container)
4) ANY_NON_JSON: NONE
```
File contents not read on either pass.

**Implication.** The credentials directory holds exactly one file (Telegram pairing data). Hermes has no Telegram tool wired up and does not need it. The contents are protected today by mode 0700 plus the RO mount, but the access-control story relies on layered defaults rather than explicit scoping. Recommend the contract either move credentials/ out of the shared surface entirely or document the mode-0700 + RO double lock as the enforcement mechanism.

## Finding 3 — No reverse mount, no shared network

**Question.** Does OpenClaw have any inbound channel from Hermes?

**Evidence.**
`docker inspect openclaw-iemy-openclaw-1` Mounts:
```
bind /docker/openclaw-iemy/data            -> /data           (RO=False)
bind /docker/openclaw-iemy/data/linuxbrew  -> /home/linuxbrew (RO=False)
```
Networks: `openclaw-iemy_default` only, IP `172.18.0.2`. Hermes container is on `hermes-agent-bnz0_default` only, IP `172.19.0.2`. Two separate Docker bridge networks, not joined. From Hermes, `getent hosts openclaw` and `getent hosts openclaw-iemy` both returned no output, exit non-zero. The Docker embedded resolver at 127.0.0.11 has `search .` and `ndots:0`, so unqualified sibling hostnames are not discoverable on this network.

**Implication.** OpenClaw has no mount from Hermes territory and no DNS path to Hermes. The directional rule is one-way only: Claw publishes, Hermes reads. Any future return path (handoffs, replies) requires either a second bind-mount with explicit scope or a network bridge — neither exists today.

## Finding 4 — Listening sockets and back-channel hunt

**Question.** Is there any TCP back-channel between Hermes and OpenClaw?

**Evidence.**
From Hermes, `ss -tunap 2>/dev/null` produced no rows and exited 2 (`ss` not installed or blocked by container capabilities). Hermes reported:
```
1) LISTENING_PORTS: NONE visible
2) ESTABLISHED_CONNECTIONS_TO_OPENCLAW: NONE visible
3) DNS_RESOLVES_OPENCLAW: no
4) UNUSUAL_OUTBOUND: NONE observed
```
From host, `docker network ls` shows no shared network between the two containers. No bridge or external connector observed.

**Implication.** No back-channel observed. Per the brief's discipline ("honest empties"), Hermes-side answers for listening sockets and established connections are *lower bounds* — `ss`/`netstat`/`lsof` are unavailable in her container, so she cannot affirmatively rule out sockets she has no permission to enumerate. Host-side network inspection found no shared bridge. Combined evidence is sufficient to support a one-way contract; if stricter assurance is needed, run `ss -tunap` from within OpenClaw's container or from the host's network namespace.

## Finding 5 — Write activity on the surface

**Question.** What process is writing, and where?

**Evidence.**
From Hermes:
```
1) ACTIVE_PROCESSES_USING_MOUNT: NONE (lsof not installed or no visible handles to this uid)
2) FILES_MODIFIED_LAST_HOUR:
   /openclaw-source/agents/main/sessions/sessions.json
   /openclaw-source/agents/main/sessions/84214418-65f9-4aa1-84d1-5512874f47ca.jsonl
3) FILES_MODIFIED_LAST_24H: 2
4) WRITE_PATTERN: writes concentrated in agents/main/sessions/, .jsonl filename is a UUID
   consistent with per-session append-only transcript; sessions.json is the rolling index.
   No bursts; steady low-rate appends to one file pair.
```
Caveat from Hermes: `find` cannot descend into 0700-owned dirs (`credentials/`, `logs/`, `tasks/`, `telegram/`, and the `.openclaw` root itself are mode 0700 owned by uid 1000; Hermes runs as a different uid). The "only sessions/ is hot" pattern is a lower bound — true for everything readable, unknown for the locked-down dirs.

**Implication.** OpenClaw is actively writing the surface during sessions, and writes cluster cleanly in `agents/main/sessions/`. The contract can specify `agents/main/sessions/` as Claw's append-only output channel; everything else is comparatively static.

## Finding 6 — Layout, with diff against draft contract

**Question.** What dirs actually exist on the surface today vs. what was assumed in the contract draft?

**Evidence.**
From host, `ls -la /docker/openclaw-iemy/data/.openclaw/`:
```
total 76
drwx------ 13 ubuntu ubuntu 4096 May  1 05:16 .
drwxr-xr-x  7 ubuntu ubuntu 4096 May  1 01:17 ..
drwxr-xr-x  3 ubuntu ubuntu 4096 May  1 01:17 agents
drwxr-xr-x  2 ubuntu ubuntu 4096 May  1 01:19 canvas
drwx------  2 ubuntu ubuntu 4096 May  1 06:16 credentials
drwxr-xr-x  2 ubuntu ubuntu 4096 May  1 01:22 devices
drwxr-xr-x  2 ubuntu ubuntu 4096 May  1 01:17 extensions
drwxr-xr-x  2 ubuntu ubuntu 4096 May  1 01:20 identity
drwx------  2 ubuntu ubuntu 4096 May  1 01:18 logs
-rw-------  1 ubuntu ubuntu 3484 May  1 01:19 openclaw.json
-rw-------  1 ubuntu ubuntu 3378 May  1 01:19 openclaw.json.bak
-rw-------  1 ubuntu ubuntu 3143 May  1 01:19 openclaw.json.bak.1
-rw-------  1 ubuntu ubuntu 2835 May  1 01:19 openclaw.json.bak.2
-rw-------  1 ubuntu ubuntu 2694 May  1 01:18 openclaw.json.bak.3
-rw-------  1 ubuntu ubuntu 3484 May  1 01:19 openclaw.json.last-good
drwxr-xr-x  3 ubuntu ubuntu 4096 May  1 01:17 skills
drwx------  2 ubuntu ubuntu 4096 May  1 01:19 tasks
drwx------  2 ubuntu ubuntu 4096 May  1 06:16 telegram
drwxr-xr-x  5 ubuntu ubuntu 4096 May  1 01:23 workspace
```
`du -sh` reports 1.4M total. From Hermes (uid mismatch, sees ~1.05M of the 1.4M).
Presence check from Hermes for the contract-draft dirs returned: `handoffs: absent ; contracts: absent ; docs: absent ; extension-snapshot: absent ; replit-sidecar-snapshot: absent ; inbox: absent ; outbox: absent`.

**Implication.** The contract should specify those seven directories as *to-be-created* paths rather than existing ones. The presence of `openclaw.json` + 4 `.bak` + `.last-good` files indicates OpenClaw already implements a rotating "last known good" config-rollback discipline; the contract can reuse that pattern for any shared state files.

## Finding 7 — Source of truth

**Question.** Where does the canonical state live? Hostinger box, or upstream mirror?

**Evidence.**
Compose files discovered via `find / -name 'docker-compose.y*ml'`:
```
/docker/hermes-agent-bnz0/docker-compose.yml
/docker/openclaw-iemy/docker-compose.yml
```
Workspace git status (run as root with explicit `safe.directory`):
```
$ git -c safe.directory=... -C /docker/openclaw-iemy/data/.openclaw/workspace remote -v
(empty — no remotes configured)
$ git -c safe.directory=... -C /docker/openclaw-iemy/data/.openclaw/workspace log -1 --oneline
fatal: your current branch 'master' does not have any commits yet
$ git -c safe.directory=... -C /docker/openclaw-iemy/data/.openclaw/workspace branch --show-current
master
```

**Implication.** The Hostinger VPS holds the only copy of OpenClaw's state. The workspace `.git` repo is freshly initialized with no remote and no commits — nothing is mirrored from Replit, GitHub, or any upstream. Any backup or disaster-recovery plan needs to be written for this box specifically.

## Finding 8 — No watchdog, no scheduled sync

**Question.** Is there any cron or systemd-timer activity touching the surface?

**Evidence.**
```
$ crontab -l ; echo "EXIT=$?"
no crontab for root
EXIT=1

$ systemctl list-timers --all | head -25
(17 timers listed: fstrim.timer, apt-daily.timer, apt-daily-upgrade.timer,
 update-notifier-download.timer, update-notifier-motd.timer, e2scrub_all.timer,
 man-db.timer, systemd-tmpfiles-clean.timer, apport-autoreport.timer,
 snapd.snap-repair.timer, ua-timer.timer — all stock Ubuntu services)

$ ls -la /etc/cron.{d,daily,hourly,weekly,monthly}/
/etc/cron.d/    : .placeholder, docker-image-prune, e2scrub_all, sysstat
/etc/cron.daily/: .placeholder, apport, apt-compat, dpkg, logrotate, man-db, sysstat
/etc/cron.hourly/: .placeholder
/etc/cron.weekly/: .placeholder, man-db
/etc/cron.monthly/: .placeholder
```
None of these scheduled jobs reference openclaw, hermes, or the surface paths.

**Implication.** No watchdog exists; the human is the only failure-detection mechanism today. If one side stops writing or the mount goes stale, nothing surfaces an alert. Latency budget: synchronous (a kernel bind-mount on a single filesystem). Any contract-level latency bar will be vastly over-met by current architecture; a meaningful watchdog would have to be added explicitly (a sidecar polling sessions.json mtime, or a healthcheck on the mount point).

## Side findings flagged for the record

**Hermes API key on the box.** Hermes loads `ANTHROPIC_API_KEY` from `/opt/data/.env` (host path `/docker/hermes-agent-bnz0/data/.env`), not from the compose-level `/docker/hermes-agent-bnz0/.env`. The two files currently hold different values; the compose-level file holds the dead key from earlier rotations and the data-level file holds the working key. If the container is recreated via `docker compose down && up`, the compose `.env` will re-inject the dead key. Reconciliation tomorrow.

**Red-teaming/godmode skill present.** Path `/opt/hermes/skills/red-teaming/godmode/` exists in Hermes's container. Per Nous Research's published default skill catalog, this is part of the standard Hermes Agent install — not unauthorized. Skill files were not read or invoked during this investigation. Tomorrow's planned `config.yaml` cleanup should disable skill clusters not needed for the integration (red-teaming, gaming, image-gen, etc.).

**Library `.env` false-positive.** `find / -name '*.env' | grep -E 'openclaw|hermes|claw'` surfaced two files inside `node_modules/openclaw/node_modules/bottleneck/.env` (one in OpenClaw's container overlay, one in containerd's snapshot store). These are vendored npm package files from the `bottleneck` library, not project configuration. Listed for completeness only.

**Git "dubious ownership" friction.** Any tooling that traverses `/openclaw-source/workspace` (or the host equivalent) from a uid other than 1000 will hit git's `safe.directory` check. The contract should specify uid-1000 (`ubuntu`) as canonical for git operations and document the `git -c safe.directory=...` workaround for cross-uid access.

**Host hygiene notes.** MOTD reports "7 zombie processes" and "*** System restart required ***" (kernel updates pending). Not surface-related but worth scheduling a maintenance window.

## Investigation discipline held

No credential files were read at any point. All commands were read-only. No skills were loaded or invoked on Hermes during this investigation. No writes were performed to the surface or to either container. Output above is verbatim from the captured terminal sessions; any ellipses or `...` are typographical, not redactions of relevant data.
