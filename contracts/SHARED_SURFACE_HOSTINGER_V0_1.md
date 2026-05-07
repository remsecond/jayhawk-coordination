---
title: Shared Coordination Surface (Hostinger) v0.1 — Telegram + Agent Bus
status: candidate
type: spec
spec_version: v0.1
owner: roberto (Tab)
authors:
  - dispatch (claude)
  - tab claude (parallel draft pending)
created: 2026-05-07
last_updated: 2026-05-07
covers:
  - Hermes/Claw coordination on Hostinger (extends v0)
related:
  - contracts/SHARED_SURFACE_HOSTINGER_V0.md
  - handoffs/2026-05-07_hermes-openclaw-integration-findings.md
note: "Candidate v0.1 spec extending v0. Where v0.1 diverges from v0, v0.1 supersedes. Tab Claude has a parallel draft in flight; this file is the starting position to diff against."
---

# Shared Coordination Surface v0.1 — Telegram + Agent Bus

> **Status: candidate.** Drafted 2026-05-07 from Roberto's locked decisions. v0 is unchanged — v0.1 sits beside it. Sections numbered §11.x to extend v0's table of contents without renumbering the prior doc.

## §11.1 Telegram

### §11.1.1 Per-agent bots

Each agent operates its own Telegram bot identity. **Not a shared bot.** Two distinct bots, two distinct tokens, two distinct usernames. Rationale: when a human sees a Telegram message, the sender identity carries authorship without parsing the body.

Token + pairing storage:

| Agent | Path on host | Visibility |
|---|---|---|
| Claw | `/docker/openclaw-iemy/data/.openclaw/credentials/telegram-pairing.json` | Claw-private (already exists) |
| Hermes | `/docker/hermes-agent-bnz0/data/credentials/telegram-pairing.json` | Hermes-private (to be created in `/opt/data/credentials/` inside her container) |

Hermes's pairing file MUST live inside her own `/opt/data` writeable volume, never on the shared surface or anywhere visible to Claw.

### §11.1.2 Private 1:1 chats only (v0.1 scope)

Each bot talks to humans in **1:1 chats only**. No group chats. No broadcast topics. No shared inbox channels. The bot DM is a per-human, per-agent thread.

Rationale: privacy + clear authorship + no ambiguity about which agent is "in" any given chat. Group chat surfaces (multiple humans + agents in one room) are explicitly out of scope and deferred to v0.2+.

### §11.1.3 Bot delivery (open question, see §11.4)

Polling vs webhook is unresolved. Polling is simpler — no public ingress required, no inbound port to expose on Hostinger. v0.1 default lean: **polling**, decided when each agent's bot wires up.

## §11.2 Agent bus

The agent bus is the bidirectional coordination channel on `/coordination/`. v0.1 tightens v0's writer rule.

### §11.2.1 Strict directional rule (replaces v0 §4)

Two outbound directories, scope enforced by uid + mount mode. **No shared rw directory.**

| Path | Originator | Receiver | Mode |
|---|---|---|---|
| `/coordination/from-claw/` | Claw (uid 1000) — **rw** | Hermes (uid 2000) — **ro** | originator-only writes |
| `/coordination/from-hermes/` | Hermes (uid 2000) — **rw** | Claw (uid 1000) — **ro** | originator-only writes |
| `/coordination/inbox/for-claw/` | humans — write | Claw — read | per-agent inbox (v0 §11.2 locked) |
| `/coordination/inbox/for-hermes/` | humans — write | Hermes — read | per-agent inbox (v0 §11.2 locked) |
| `/coordination/contracts/` | repo sync — write | both — read | snapshot, runtime read-only |

**v0's `/coordination/shared/` dir is removed.** v0 allowed both agents to write into a shared dir using uuid-scoped filenames. v0.1 forbids it: each agent writes only its own outbound dir. Stricter, less ambiguity, no possibility of two writers claiming the same path.

### §11.2.2 No cross-writes

Receivers never write to the originator's dir, including for acknowledgement or reply. Replies go in the receiver's own outbound dir, by convention referencing the original message id (see §11.4 — message id convention is open).

### §11.2.3 Permission model on disk

Enforced by uid + Linux DAC, not by application-level convention:

```
/coordination/from-claw/      drwxr-xr-x  1000 (claw)    coordination (gid 3000)
/coordination/from-hermes/    drwxr-xr-x  2000 (hermes)  coordination (gid 3000)
```

Each outbound dir is owned by its originator's uid with mode `0755` — receiver can list+read, only originator can write. Group `coordination` exists for read traversal but does not grant write.

### §11.2.4 Verification

Bus correctness is validated post-deploy via the verification block in `docs/HOSTINGER_DEPLOYMENT_NOTES.md` §7 commands #6 and #7 (cross-visibility + write-permission tests). v0.1 inherits those tests unchanged.

## §11.3 Conditional credentials migration

This section replaces "open question 5" in v0 (telegram pairing exposure) with a **trigger-conditioned migration plan**, not deferred work.

### §11.3.1 Current posture (v0, preserved)

`telegram-pairing.json` stays at `.openclaw/credentials/`. It is **not** moved into a `.private/` sibling and **not** added to the shared surface. The v0 allowlist on Hermes's compose structurally hides the entire `credentials/` directory from her view.

This posture relies on **layered defaults**:

- Mount mode: ro from Hermes's side.
- File mode: 0600 uid 1000.
- Allowlist: `credentials/` is not in Hermes's compose mounts.
- Hermes uid: 2000 (after v0 deploy), DAC blocks reads even if a path leaked.

While all four hold, the posture is sufficient. The contract does not pretend it is sufficient unconditionally.

### §11.3.2 Migration triggers

When **any** of the following occurs, migration to a dedicated private credentials dir becomes **required**, not optional. Triggers split into two classes:

**Defense-degradation triggers** (a defensive layer weakens):

1. **Hermes uid regresses** away from the dedicated 2000:2000 (e.g., back to root or uid 1000) — invalidates the DAC defense layer.
2. **The OpenClaw → Hermes bind mount flips to rw** for any reason — invalidates kernel-level write protection and changes the threat model.
3. **A new credential file lands** in `.openclaw/credentials/` *or* anywhere else on the shared surface — the assumption "the only thing in credentials/ is one Telegram pairing file Hermes already can't read" stops holding.

**Scope-expansion triggers** (the threat surface grows):

4. **A third agent joins the coordination surface.** Two-agent posture relies on knowing exactly which uid is on the other side; a third uid invalidates the per-uid allowlist reasoning.
5. **Off-box deployment enters the roadmap.** Multi-host coordination removes the "single Linux kernel enforces ro" guarantee and introduces network transports — a different threat model that requires structural credential isolation, not allowlist-on-mount.

The contract surfaces all five triggers explicitly so that the moment to act is documented, not discovered.

### §11.3.3 Migration definition (when triggered)

1. Create `/docker/openclaw-iemy/data/.private/credentials/` on the host. Mode 0700 uid 1000.
2. Move all credential files from `.openclaw/credentials/` into `.private/credentials/`.
3. Update OpenClaw's container compose to bind `/docker/openclaw-iemy/data/.private/credentials → /data/.openclaw/credentials` rw. Claw's internal view is unchanged.
4. Confirm no compose mount on **any** container references `.private/`.
5. Re-run the verification block in deployment notes §7 to confirm the credentials path is invisible to Hermes.
6. Document the migration in a new handoff dated to the trigger event, naming which trigger fired.

### §11.3.4 Why conditional, not preemptive

Preemptive migration would add complexity (new bind, new compose path) for no gain while the four defenses still stack. Conditional migration preserves simplicity now and forces correctness on the day a defense weakens. The trigger conditions live in the contract so the requirement isn't human-memory-dependent.

## §11.4 Open questions (v0.1.x or later)

1. **Hermes private credential path inside container.** v0.1 specifies `/opt/data/credentials/` — confirm with Hermes's actual filesystem layout when wired.
2. **Bot delivery mechanism.** Polling (default lean) vs webhook. Polling avoids public ingress but uses long-poll connections; webhook needs a publicly resolvable URL. Decide per-agent.
3. **Group chat surface.** Out of scope for v0.1. Specify when it matters (probably v0.2 or v0.3).
4. **Bus message id convention.** ISO timestamp + agent prefix? UUID? Content hash? Affects reply-by-reference. v0.1.x decision.
5. **Bus garbage collection.** Outbound dirs can grow without bound. Time-based prune (delete files >30d)? Ring buffer (last N files)? Manual? v0.1.x decision.
6. **Pairing-state visibility (replaces v0 Q5).** Should Hermes ever see *redacted* Telegram pairing state from Claw (e.g., "paired: true, identity: Tab")? v0.1 default: no — pairing state is private. Revisit if a use case appears.

---

*End of Shared Coordination Surface v0.1 candidate. Diff against Tab Claude's parallel draft when it lands; reconcile any divergence as a v0.1.1 revision.*
