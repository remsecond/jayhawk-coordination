---
title: Rehydration Packet — Hermes ↔ OpenClaw Coordination
status: reference
type: index
created: 2026-05-07
purpose: "Single self-contained paste-ready document to rehydrate any Claude session that lost context on the Hermes↔OpenClaw shared-surface work. Lossless re-bootstrap."
---

# Rehydration Packet — Hermes ↔ OpenClaw Coordination

> **Use:** paste this entire document into a Claude session that needs context on the Hermes/OpenClaw integration on Hostinger. Self-contained — no external links required to act, but linked artifacts on the branch are the canonical sources if discrepancies appear.

## 0. Where everything lives

- **Repo:** `remsecond/jayhawk-coordination` (GitHub)
- **Branch:** `claude/review-openclaw-sessions-BNPso`
- **PR:** not opened yet — Roberto (Tab) is holding until the deploy lands and §7 verification passes.

Files on the branch (as of 2026-05-07):

```
CLAUDE.md                                 auto-loaded at session start; orientation
contracts/
  SHARED_SURFACE_HOSTINGER_V0.md          v0 candidate (11 sections)
  SHARED_SURFACE_HOSTINGER_V0_1.md        v0.1 candidate (Telegram + bus + conditional credentials)
docs/
  HOSTINGER_DEPLOYMENT_NOTES.md           executable deploy plan; §7 = post-deploy verification block
  SESSION_END_CHECKLIST.md                three-step wrap-up
handoffs/
  2026-05-07_hermes-recon-shared-surface.md
  2026-05-07_hermes-openclaw-integration-findings.md
  2026-05-07_hostinger-cleanup-next-steps.md
  2026-05-07_hermes-skill-prune-candidate.md
  2026-05-07_for-claw-telegram-research.md
  2026-05-07_rehydration-packet.md        ← this file
outbox/
  README.md                               ad-hoc artifacts; mirrored to /coordination/from-jayhawk-repo/
```

## 1. The two boxes (load-bearing facts)

- Single Hostinger VPS: `srv1317315.hstgr.cloud` (76.13.118.164).
- Two containers on it: `openclaw-iemy-openclaw-1` (Claw, IP 172.18.0.2, owned uid 1000 `ubuntu`) and `hermes-agent-bnz0-hermes-agent-1` (Hermes, IP 172.19.0.2, currently runs as root or uid 1000 — to be changed to uid 2000 in deploy).
- Compose files: `/docker/openclaw-iemy/docker-compose.yml` and `/docker/hermes-agent-bnz0/docker-compose.yml`. Both local, no upstream mirror. Hostinger box is canonical.
- Today's shared surface: **single** kernel bind-mount in Hermes's compose: `/docker/openclaw-iemy/data/.openclaw → /openclaw-source` ro. Same disk, same fs, kernel-enforced ro on Hermes side.
- Two compose-default networks (`openclaw-iemy_default`, `hermes-agent-bnz0_default`), **not joined**. No DNS between containers. No back-channel.
- Today: one-way only. Claw publishes; Hermes reads. Hermes has no return path.

## 2. v0 contract — section list

`contracts/SHARED_SURFACE_HOSTINGER_V0.md`. Status: candidate.

| § | Heading | Summary |
|---|---|---|
| 1 | Scope and goal | Two-way coordination for Claw + Hermes; credentials physically isolated |
| 2 | Surfaces | Three surfaces: Claw publish (ro), Claw private (never crosses), Coordination (rw both) |
| 3 | Public/private separation — allowlist on Hermes's compose | Per-subdir ro mounts — `agents`, `canvas`, `devices`, `extensions`, `skills`, `tasks`, `workspace`, `openclaw.json`. NOT `credentials`, `telegram`, `identity`, `logs`. Replaces today's whole-tree mount. |
| 4 | Coordination surface layout | New `coordination-shared` host bind at `/coordination` rw both sides. Subtrees scoped per writer. |
| 5 | Identity and permissions | Claw uid 1000:1000; **Hermes changes to 2000:2000**; gid 3000 `coordination` shared. |
| 6 | Git access on the shared surface | Every agent runs `git config --global --add safe.directory` for shared paths at startup. |
| 7 | **Contract sync** | `/coordination/contracts/` is a snapshot of the jayhawk-coordination repo's `contracts/` dir, pinned by commit hash. Read-only at runtime. **Note: this is contract §7, NOT the deployment-notes §7 verification block — see §3 of this packet.** |
| 8 | Network model | Compose networks not joined; coordination is filesystem-only by design. |
| 9 | Failure modes and rules | Mount-missing = refuse to start. Stale data = check mtime. No watchdog in v0 (Tab is HITL). |
| 10 | Out of scope (v0) | Multi-host, network IPC, agents beyond Claw+Hermes, encryption-at-rest. |
| 11 | Open questions | See §6 of this packet. |

## 3. Deployment notes — §7 verification block (the 9 commands)

`docs/HOSTINGER_DEPLOYMENT_NOTES.md` §7. Run these from inside the containers AFTER the deploy lands. All nine must pass before the surface is considered live.

```
# 1. Allowlist worked: 7 dirs + openclaw.json under /openclaw-source.
docker exec hermes-agent-bnz0-hermes-agent-1 ls -la /openclaw-source
# Expected: agents canvas devices extensions skills tasks workspace openclaw.json
# Expected ABSENT: credentials telegram identity logs

# 2. Credentials gone from Hermes view.
docker exec hermes-agent-bnz0-hermes-agent-1 ls /openclaw-source/credentials 2>&1
# Expected: "No such file or directory"

# 3. Hermes is uid 2000.
docker exec hermes-agent-bnz0-hermes-agent-1 id
# Expected: uid=2000 ... gid=2000 ... groups=2000,3000(coordination)

# 4. Coordination volume rw from Hermes.
docker exec hermes-agent-bnz0-hermes-agent-1 sh -c \
  'echo test > /coordination/from-hermes/_writetest && rm /coordination/from-hermes/_writetest'

# 5. Coordination volume rw from Claw.
docker exec openclaw-iemy-openclaw-1 sh -c \
  'echo test > /coordination/from-claw/_writetest && rm /coordination/from-claw/_writetest'

# 6. Cross-visibility: Claw reads what Hermes wrote.
docker exec hermes-agent-bnz0-hermes-agent-1 sh -c 'echo from-hermes-msg > /coordination/from-hermes/handshake.txt'
docker exec openclaw-iemy-openclaw-1 cat /coordination/from-hermes/handshake.txt
# Expected: "from-hermes-msg"

# 7. Hermes cannot WRITE to from-claw/.
docker exec hermes-agent-bnz0-hermes-agent-1 sh -c 'echo x > /coordination/from-claw/_should-fail' 2>&1
# Expected: Permission denied

# 8. Hermes cannot WRITE to /openclaw-source.
docker exec hermes-agent-bnz0-hermes-agent-1 sh -c 'echo x > /openclaw-source/openclaw.json' 2>&1
# Expected: Read-only file system

# 9. git safe.directory works.
docker exec hermes-agent-bnz0-hermes-agent-1 git -C /openclaw-source/workspace status 2>&1
# Expected: status output (or "no commits yet"), NOT "dubious ownership"
```

If all nine pass, the surface meets the contract. Roberto's directive: parked v0 §11 questions get decided **after** §7 passes — he wants behavior data before shape decisions.

## 4. v0.1 contract — section list (extends v0)

`contracts/SHARED_SURFACE_HOSTINGER_V0_1.md`. Status: candidate.

| § | Heading | Summary |
|---|---|---|
| 11.1.1 | Per-agent Telegram bots | Two bots, two tokens. Claw token at `.openclaw/credentials/telegram-pairing.json` (existing). Hermes token at `/docker/hermes-agent-bnz0/data/.private/hermes-telegram.token` (new). |
| 11.1.2 | 1:1 chats only | No group chats in v0.1. Avoids `privacy mode` gotcha. |
| 11.1.3 | **Long-polling REQUIRED**, webhook forbidden | `deleteWebhook` at startup MANDATORY. |
| 11.1.4 | Update offset persistence | Claw: `telegram/update-offset-default.json` (already correct). Hermes: `/opt/data/telegram-offset.json`. **Persist BEFORE acting.** |
| 11.1.5 | Idempotency | Key = `(chat_id, message_id)`. |
| 11.1.6 | **Agent↔agent over Telegram is forbidden** | Rate limits force it. Use the filesystem bus for Claw↔Hermes. Telegram is human-channel only. |
| 11.1.7 | Message format | Plain text default. HTML if formatting needed. **MarkdownV2 forbidden by default.** |
| 11.1.8 | File size limits | 50 MB out / 20 MB in. Push large artifacts via filesystem, not Telegram. |
| 11.1.9 | Library | Telegraf or grammY (Node), python-telegram-bot (Python). Raw HTTP forbidden. |
| 11.1.10 | Single-instance rule | Never run two processes with the same bot token. |
| 11.2.1 | Agent bus — strict directional rule | Two outbound dirs: `from-claw/` (Claw rw, Hermes ro), `from-hermes/` (Hermes rw, Claw ro). **No shared rw dir.** |
| 11.2.2 | No cross-writes | Replies go in receiver's own outbound, ref'ing the original message id. |
| 11.2.3 | Permission model | uid + gid + mode 0755. Group `coordination` (3000) for read traversal. |
| 11.2.4 | Verification | Inherits deploy notes §7 commands #6 and #7. |
| 11.3.1 | Conditional credentials migration — current posture | `telegram-pairing.json` stays put. Layered defaults (mount mode + file mode + allowlist + uid) are sufficient *until* a trigger fires. |
| 11.3.2 | Migration triggers (5 total) | Defense-degradation: (1) Hermes uid regresses, (2) mount flips rw, (3) new credential lands. Scope-expansion: (4) third agent joins, (5) off-box deployment on roadmap. |
| 11.3.3 | Migration definition | Move to `/docker/openclaw-iemy/data/.private/credentials/`. Bind into Claw at the same internal path. Verify Hermes invisibility. |
| 11.3.4 | Why conditional, not preemptive | Layered defaults hold today; complexity for no gain unless a defense weakens. |
| 11.4 | Open questions | See §6 of this packet. |

## 5. Finding 2 — the either-or framing and its resolution

**Finding 2 (verbatim, from the rendered findings doc) — credentials directory exposure**

> The credentials directory holds exactly one file (Telegram pairing data). Hermes has no Telegram tool wired up and does not need it. The contents are protected today by mode 0700 plus the RO mount, but the access-control story relies on layered defaults rather than explicit scoping. **Recommend the contract either move credentials/ out of the shared surface entirely or document the mode-0700 + RO double lock as the enforcement mechanism.**

**Resolution (Roberto's call, 2026-05-07):**

> Move credentials/ out, via per-subdir allowlist on Hermes's compose. The "double lock" is fail-open — recon showed Hermes could `ls` 0700 uid-1000 dirs (because she runs as root or uid 1000), so the lock isn't actually locked.

Allowlist landed in v0 §3. Reasoning: structural absence beats permission-based defaults. Five migration triggers in v0.1 §11.3.2 force the next-level migration when any layered default breaks.

## 6. Open questions still parked (decided after §7 passes)

From v0 §11 (some now resolved, some still open):

1. ~~OpenClaw data-dir relocation.~~ Resolved — allowlist on Hermes compose, Claw unchanged.
2. **`inbox/` shape** — single dir or per-agent. **Decided 2026-05-07: per-agent (`inbox/for-claw/`, `inbox/for-hermes/`).**
3. **Heartbeat cadence.** Recommendation 30s; not finalized. Deferred behind §7 behavior data.
4. Need a `from-<agent>/version.json` schema declaration?
5. ~~Telegram pairing exposure.~~ Resolved — strictly Claw-private; Hermes has her own token in `.private/`.
6. Should jayhawk-coordination repo itself accept agent pushes for history survival? Default: no in v0.
7. Does Claw need to react to `from-hermes/` writes within seconds (fs-watcher) or event-driven-on-next-task? Default: no watcher in v0.
8. **Skill prune scope.** Tomorrow's `config.yaml`-equivalent cleanup. Hermes startup banner showed 28 tools / 75 skills. Filesystem-based, not YAML — see `handoffs/2026-05-07_hermes-skill-prune-candidate.md`.

From v0.1 §11.4 (Telegram-side):

3. Group chat surface — out of scope for v0.1.
4. Bus message-id convention — UUID? ISO timestamp? Content hash?
5. Bus garbage collection — time-based prune? Ring buffer?
6. Pairing-state visibility — Hermes never sees Claw's, by default.
7. Hermes Node-vs-Python stack — affects library choice in §11.1.9.

## 7. The "three offers" + Roberto's calls

From an earlier exchange in Tab Claude's thread, before the §11 work began. Tab Claude offered three additions; Roberto picked:

| Offer | Roberto's call |
|---|---|
| Re-run the recon for fresh audit-trail timestamp | **No** — 2026-05-07 capture is good |
| Add 9th/10th finding for the duplicated-tab Hermes UI artifact | **No** — cosmetic ttyd artifact, not surface concern |
| Add a "next-steps" section for tomorrow's `config.yaml` prune | **Yes** — landed at `handoffs/2026-05-07_hostinger-cleanup-next-steps.md` (handoffs/, not docs/) |

## 8. Other locked decisions (2026-05-07)

| Decision | Pick | Rationale |
|---|---|---|
| Credentials handling | Allowlist on Hermes compose (v0 §3) | Allowlist is structural; double-lock is fail-open |
| Inbox shape | Per-agent (`inbox/for-claw/`, `inbox/for-hermes/`) | Distinct roles → addressed routing |
| Bot delivery | Long-polling REQUIRED, webhook forbidden | No public ingress; matches existing egress posture |
| Agent↔agent transport | Filesystem bus only, never Telegram | Rate-limit math forces it |
| Hermes uid | 2000:2000 + gid 3000 (`coordination`) | Defense-in-depth; allowlist + DAC |
| Home Assistant skill toolset | Keep | "no but may" — preserve optionality |
| TTS skill | Prune | "no idea" → trivially restorable |
| Vision skill | Keep | Screenshot debugging |
| image_gen toolset | Prune | Not coordination-relevant |
| Truncated skill clusters | Item-by-item review when filesystem listing arrives | Default keep unless specific reason to prune |

## 9. What still needs to happen

1. **Deploy** per `docs/HOSTINGER_DEPLOYMENT_NOTES.md`. Allowlist on Hermes compose, new `coordination-shared` volume, Hermes uid 2000, git safe.directory entrypoint.
2. **§7 verification** — run all 9 commands. All must pass.
3. **Decide parked v0 §11 questions 3, 4, 6, 7, 8** based on observed behavior post-deploy.
4. **Skill prune** — get a `ls /opt/data/skills` listing, categorize per `handoffs/2026-05-07_hermes-skill-prune-candidate.md`, propose removal plan, Roberto approves, operator applies.
5. **Telegram bot wire-up** for Hermes per v0.1 §11.1 (long-polling, offset persistence, idempotency, library choice TBD on stack).
6. **Open the PR** when (1)-(2) are clean.

## 10. Pointers for further detail

If anything in this packet is ambiguous, the canonical sources on the branch are:

- v0 contract: `contracts/SHARED_SURFACE_HOSTINGER_V0.md`
- v0.1 contract: `contracts/SHARED_SURFACE_HOSTINGER_V0_1.md`
- Deploy plan: `docs/HOSTINGER_DEPLOYMENT_NOTES.md`
- Recon raw: `handoffs/2026-05-07_hermes-recon-shared-surface.md`
- Findings doc: `handoffs/2026-05-07_hermes-openclaw-integration-findings.md`
- Cleanup tasks: `handoffs/2026-05-07_hostinger-cleanup-next-steps.md`
- Skill prune candidate: `handoffs/2026-05-07_hermes-skill-prune-candidate.md`
- Telegram research: `handoffs/2026-05-07_for-claw-telegram-research.md`

---

*End of rehydration packet. Anyone with this document plus the canonical sources on the branch can pick up the work without further chat-thread context.*
