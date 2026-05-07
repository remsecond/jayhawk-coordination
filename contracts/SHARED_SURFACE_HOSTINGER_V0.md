---
title: Shared Coordination Surface (Hostinger) v0
status: candidate
type: spec
spec_version: v0
owner: roberto (Tab)
authors:
  - dispatch (claude)
created: 2026-05-07
last_updated: 2026-05-07
covers:
  - Hostinger host running OpenClaw + Hermes containers
  - the bind-mounted filesystem surface they coordinate over
related:
  - docs/HOSTINGER_DEPLOYMENT_NOTES.md
  - handoffs/2026-05-07_hermes-recon-shared-surface.md
note: "Documentation only. No deploy actions follow from filing this. The deployment notes file is the executable counterpart."
---

# Shared Coordination Surface (Hostinger) v0

> **Status: candidate.** Names the rules under which OpenClaw ("Claw") and Hermes coordinate on Hostinger via shared filesystem. Two containers, one host, no network IPC between them today.

## 1. Scope and goal

Two agents run as separate containers on the same Hostinger host:

- `openclaw-iemy` — OpenClaw runtime (writes session state, runs skills, owns Telegram pairing + identity)
- `hermes-agent-bnz0` — Hermes runtime (reads what Claw publishes, must be able to signal back)

**Goal:** a deterministic, auditable, two-way coordination channel between them, with credentials physically isolated from the cross-container surface.

**Non-goal (v0):** network IPC, multi-host federation, agents beyond Claw and Hermes. Add as v0.1+.

## 2. Surfaces

Three filesystem surfaces. Each has one purpose and one owner.

| Surface | Path (in each container) | Mode | Owner | Purpose |
|---|---|---|---|---|
| **Claw publish** | `/openclaw-source` (Hermes) | ro | Claw writes via its own rw view | Hermes observes Claw's runtime state (sessions, tasks, workspace) |
| **Claw private** | not mounted into Hermes | n/a | Claw only | credentials, telegram pairing, identity, logs — never crosses |
| **Coordination** | `/coordination` (both) | rw | both, scoped by subdir | bidirectional signaling channel |

This is a refactor of today's setup, where Claw's *whole* `.openclaw/` dir is bind-mounted into Hermes — exposing `credentials/` and `telegram/` even though file mode 0600 happens to block reads. v0 makes the boundary structural, not permission-based.

## 3. Public/private separation — allowlist on Hermes's compose

Rather than restructuring OpenClaw's data directory on the host, the public/private split is enforced by **explicit per-subdir bind mounts in Hermes's `docker-compose.yml`**. OpenClaw's data layout is unchanged.

**Public set (mounted ro into Hermes at `/openclaw-source/<subdir>`):**

| Host path | Hermes path | Mode |
|---|---|---|
| `/docker/openclaw-iemy/data/.openclaw/agents` | `/openclaw-source/agents` | ro |
| `/docker/openclaw-iemy/data/.openclaw/canvas` | `/openclaw-source/canvas` | ro |
| `/docker/openclaw-iemy/data/.openclaw/devices` | `/openclaw-source/devices` | ro |
| `/docker/openclaw-iemy/data/.openclaw/extensions` | `/openclaw-source/extensions` | ro |
| `/docker/openclaw-iemy/data/.openclaw/skills` | `/openclaw-source/skills` | ro |
| `/docker/openclaw-iemy/data/.openclaw/tasks` | `/openclaw-source/tasks` | ro |
| `/docker/openclaw-iemy/data/.openclaw/workspace` | `/openclaw-source/workspace` | ro |
| `/docker/openclaw-iemy/data/.openclaw/openclaw.json` | `/openclaw-source/openclaw.json` | ro |

**Private set (NEVER mounted into Hermes):** `credentials/`, `telegram/`, `identity/`, `logs/`. Plus `openclaw.json.bak*` and `openclaw.json.last-good` (rotation history is Claw-internal).

The current "everything bind-mounted" config (`/docker/openclaw-iemy/data/.openclaw → /openclaw-source ro`) is **replaced** by the explicit allowlist. What's not in the allowlist isn't visible to Hermes at all — failure mode is "missing path," not "permission denied," which is structurally fail-safe.

This approach (a) requires zero changes to OpenClaw's container or data layout, (b) makes the allowlist a single audit point in Hermes's compose, (c) survives container recreation by virtue of being declarative.

## 4. Coordination surface layout (rw, both sides)

A new docker volume `coordination-shared` is mounted rw at `/coordination` in **both** containers:

```
/coordination/
  from-claw/                 # ONLY Claw writes; Hermes reads
  from-hermes/               # ONLY Hermes writes; Claw reads
  shared/                    # both write, but only to uuid-scoped or agent-prefixed filenames
  inbox/                     # human (Tab) drops things here; agents read
  contracts/                 # snapshot of jayhawk-coordination repo's contracts/, pinned by commit hash
  README.md                  # this contract's terse summary, for in-situ reference
```

**Writer rule (load-bearing):**

- An agent MAY write only inside its own `from-<agent>/` tree, OR to `shared/<agent>-<uuid>.<ext>` files in `shared/`.
- An agent MUST NOT modify another agent's `from-<other>/` tree.
- An agent MUST NOT delete or in-place-edit any file it didn't create. Append-only or new-file semantics.
- `inbox/` is human-write, agent-read. Agents MAY move processed items to `from-<agent>/processed-inbox/` as a record, but MUST NOT delete from `inbox/` itself.
- `contracts/` is read-only at runtime; updates land via a separate sync from the jayhawk-coordination git repo (see §7).

This avoids lock contention, makes provenance trivial (the *path* tells you who wrote it), and makes audits / rollbacks possible without filesystem-level locking.

## 5. Identity and permissions

| Container | uid:gid | Effective rights |
|---|---|---|
| `openclaw-iemy` | 1000:1000 | rw on `.openclaw-public/`, rw on `.openclaw-private/`, rw on `/coordination/from-claw/`, rw on `/coordination/shared/` (for own files), ro on `/coordination/from-hermes/` |
| `hermes-agent-bnz0` | **2000:2000** (changed from current) | ro on `/openclaw-source`, rw on `/coordination/from-hermes/`, rw on `/coordination/shared/` (for own files), ro on `/coordination/from-claw/`, ro on `/coordination/inbox/` |

**Why the uid change:** today's recon shows Hermes can `ls` a `0700 uid:1000` directory, which means Hermes runs as root or uid 1000 — both wrong. Dropping Hermes to a dedicated uid (2000) makes Linux DAC the *second* line of defense if a future config change accidentally exposes something sensitive. The structural split in §3 is the first line; uid separation is belt + suspenders.

`/coordination` itself is owned `root:coordination` mode `0775`, with both container uids added to gid `coordination`. Per-subdir owners enforce the writer rule:

```
/coordination/                drwxrwxr-x  root         coordination
/coordination/from-claw/      drwxr-xr-x  1000 (claw)  coordination
/coordination/from-hermes/    drwxr-xr-x  2000 (hermes) coordination
/coordination/shared/         drwxrwxr-x  root         coordination
/coordination/inbox/          drwxrwxr-x  root         coordination   # human + both agents read; humans write
/coordination/contracts/      drwxr-xr-x  root         coordination   # synced from repo; runtime read-only
```

## 6. Git access on the shared surface

**Rule:** every agent on the shared surface MUST be able to use `git` against any `.git` directory it can see, without requiring writes to the source mount.

Recon facts: `/openclaw-source/workspace/.git` is a freshly-initialized repo on `master`, **no remotes, no commits** — it is not a real source today. But Hermes still hits `fatal: detected dubious ownership` because the repo is uid 1000 and she is not. Two-part rule so this works now and stays working when the repo gains content:

a) **Each agent's container entrypoint runs, on first start:**
```sh
git config --global --add safe.directory '/openclaw-source/workspace'
git config --global --add safe.directory '/coordination/*'
```
This writes to the agent's own `~/.gitconfig` — never to a ro mount.

b) **One canonical uid for git writes.** Only Claw (uid 1000) writes to `workspace/.git`. Any other agent or admin tool that needs to write history goes through Claw, OR pushes to a separate remote. v0 does not specify *which* remote; that's deferred until the workspace repo gains content worth pushing.

## 7. Contract sync

`/coordination/contracts/` mirrors the `contracts/` dir of the jayhawk-coordination git repo, pinned by commit hash recorded in `/coordination/contracts/.commit`. Sync runs on container start (and on demand) via `git archive` from the public coordination remote — no live working tree, just a snapshot. Rationale: agents at runtime read a known immutable snapshot; the repo remains the source of truth.

## 8. Network model

The two containers run on separate compose-default networks (`hermes-agent-bnz0_default` 172.19.0.0/16; `openclaw-iemy_default` 172.18.0.0/16) and **are not joined**. There is no DNS resolution between them, no shared IPC. v0 keeps it that way — coordination is filesystem-only. If a future need for live IPC arises, joining the networks (or a third dedicated `coordination-net`) is a one-line compose change, but is explicitly out of scope here.

## 9. Failure modes and rules

- **`/coordination` mount missing on either side**: that agent MUST refuse to start until the mount is present. No degraded mode.
- **`/openclaw-source` mount missing on Hermes**: Hermes MAY start but MUST log a startup warning and refuse any operation that depends on observing Claw state.
- **Stale data**: an agent reading another's `from-<x>/` MUST NOT assume freshness without checking mtime. There is no liveness signal in v0 — agents heartbeat into their own `from-<agent>/heartbeat.json` on a cadence to be defined in v0.1.
- **Concurrency**: two writers to the same path is a contract violation, not a runtime error to handle. The writer rule (§4) makes this structurally impossible if followed.
- **No watchdog**: Tab is the human-in-the-loop today. No cron, no timer, no out-of-band sync touches the surface. v0.1 may add a heartbeat checker that posts to `inbox/` on missed beats.

## 10. Out of scope (v0)

- Cross-host federation (multi-Hostinger or hybrid cloud).
- Network IPC (gRPC, HTTP, message queues) between containers.
- Agents beyond Claw and Hermes.
- A coordination-side ACL system. Linux DAC + path scoping is enough for v0.
- Encryption-at-rest on `/coordination`. Add when sensitive data legitimately needs to live there.

## 11. Open questions

1. ~~OpenClaw data-dir relocation.~~ **RESOLVED 2026-05-07 by Phase-2 recon:** OpenClaw's data dir is unchanged. The public/private split is enforced via per-subdir allowlist mounts in Hermes's compose (§3).
2. Should `inbox/` be a single dir or per-agent (`inbox/for-claw/`, `inbox/for-hermes/`)?
3. Heartbeat cadence — minutes? seconds? Defer to v0.1.
4. Do we need an explicit `from-claw/version.json` and `from-hermes/version.json` declaring schema version of what each agent writes?
5. Does Tab want Telegram pairing exposed to Hermes intentionally, or stay strictly Claw-private? v0 default: strictly private (`telegram/` is in the deny set, not the allowlist).
6. Should the jayhawk-coordination repo (this one) be writable by the agents (push) so coordination history survives container reset? Or is `/coordination` ephemeral-but-persistent-volume sufficient?
7. Does Claw need to read `/coordination/from-hermes/` on a cadence, or only event-driven? Affects whether a fs-watcher sidecar is needed.
8. **Hermes skill posture.** The default Nous install ships `/opt/hermes/skills/red-teaming/godmode/` and other skill clusters Hermes does not need for coordination. Tomorrow's planned `config.yaml` cleanup should prune red-teaming, gaming, image-gen, etc. The contract should reference whatever pruned set lands as the agreed allowlist. Not blocking v0.

---

*End of Shared Coordination Surface (Hostinger) v0. Candidate. Pair with `docs/HOSTINGER_DEPLOYMENT_NOTES.md` for the executable migration steps.*
