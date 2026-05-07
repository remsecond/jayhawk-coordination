# Hostinger Cleanup — Next Steps

> Captured verbatim from Tab Claude's next-steps doc, forwarded by Roberto on 2026-05-07. Pair with `handoffs/2026-05-07_hermes-openclaw-integration-findings.md` (the findings) and `contracts/SHARED_SURFACE_HOSTINGER_V0_1.md` (the v0.1 spec).

**Source:** Hermes ↔ OpenClaw integration investigation, 2026-05-07.
**Scope:** Items surfaced as remediation, hardening, or build work during the Phase 1 + Phase 2 recon. Findings are recorded separately in `system/hermes_openclaw_integration_findings.md`.

## 1. Reconcile Hermes's two .env files (priority: this week)

The container loads `ANTHROPIC_API_KEY` from `/opt/data/.env` (host path `/docker/hermes-agent-bnz0/data/.env`). The compose-level `/docker/hermes-agent-bnz0/.env` still holds the previous (dead) key value. Today this is harmless because Hermes ignores the compose-level file at runtime, but a `docker compose down && up` would re-inject the dead key and reproduce the 401 we resolved on 2026-05-07. Reconciliation: copy the live key value into the compose-level `.env` so the two files agree, or remove the compose-level `ANTHROPIC_API_KEY` assignment entirely so only the runtime location holds the secret.

## 2. Aggressive skill prune (priority: next session)

Hermes ships with the full Nous Research default catalog (28 tools, 75 skills) including red-teaming/godmode, gaming (minecraft-modpack-server, pokemon-player), media generation (gif-search, image_generate, audiocraft-audio-generation, axolotl), and other clusters not load-bearing for OpenClaw integration. Pruning these reduces the prompt-injection surface, the credential-exposure surface, and Hermes's startup time. The prune is a `config.yaml` edit inside the container; the diff itself can't be drafted until we read the current config. Action: in the next session, ask Hermes to print her `config.yaml`, then propose the disable list for review before applying.

## 3. Stand up the second bind-mount (priority: when §11 lands)

The current shared surface is one-way: Hermes can read OpenClaw's `.openclaw` directory; OpenClaw has no inbound channel. The agreed messaging architecture (per chat decision) requires Hermes to be able to write back. Implementation: create a new host directory (working name `/docker/shared/handoffs/`, ownership and permissions TBD by §11), bind it RW into Hermes and RO into OpenClaw. Edit both compose files, restart both containers. The mount is the simplest piece; the schema and idempotency discipline that runs on top of it is §11's job to specify.

## 4. Watchdog (priority: after §11)

The investigation confirmed there is no scheduled job, systemd timer, or cron task watching the surface. If OpenClaw stops writing `sessions.json`, no alert fires; the human is the only failure detector. A small systemd timer (or a Hermes cronjob skill, which she has in her tool list) checking the mtime of `agents/main/sessions/sessions.json` against a freshness threshold and posting a Telegram message when it stalls would close that gap. Implementation can be deferred until §11 lands the Telegram bot.

## 5. Host hygiene (priority: scheduled maintenance window)

The MOTD reports 7 zombie processes and "*** System restart required ***" (kernel updates pending). Neither is integration-related, but the box hasn't been rebooted in a while. A reboot during a scheduled window will clear both and let us re-verify both containers come up cleanly with the working API key. Coordinate with whatever conversation traffic is in flight.

## 6. Credentials hygiene (priority: deferred — no action tonight)

Per chat decision, the existing `telegram-pairing.json` stays where it is (mode 0700 protects it from Hermes's uid; the RO mount provides defense-in-depth). New credentials added going forward — including any Hermes Telegram bot token — go to a private path on the agent's own host volume, not into the shared `credentials/` directory. The migration of existing credentials to a fully separate `/docker/openclaw-iemy/data/.private/credentials/` tree is deferred until either a third agent appears or off-box deployment is on the roadmap.

## 7. Git safe.directory documentation (priority: low, document only)

The `/docker/openclaw-iemy/data/.openclaw/workspace` repo is owned uid 1000 (ubuntu). Any tooling that runs git operations from a different uid (root, Hermes, an admin agent) will hit "fatal: detected dubious ownership" until a safe.directory exception is configured. Document in the contract or operations runbook that uid 1000 is canonical for git operations, and that other accessors must use `git -c safe.directory=/path` for read-only operations.
