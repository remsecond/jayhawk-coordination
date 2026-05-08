---
title: Secrets recon — Hostinger /opt/data
date: 2026-05-07
status: open
type: infrastructure_cleanup
case: moyer
---

# Secrets recon — 2026-05-07 evening

Recon ran by the Hermes/Hostinger agent on /opt/data and adjacent paths. Read-only inventory, no values printed except one accidental partial-key fragment (see issue 4 below).

## What was found

**Config:** `/opt/data/config.yaml` — 56 bytes, provider + default model only. No secrets.

**Live env file:** `/opt/data/.env` — 237 B, mode 600, root:root. Contains ANTHROPIC_API_KEY.

**Backup env files (issue):**
- `/opt/data/.env.bak.1778099437` — 129 B, mode 644 — world-readable
- `/opt/data/.env.bak.preFix.1778142256` — 129 B, mode 644 — world-readable
- Both contain ANTHROPIC_API_KEY at relaxed permissions.

**Credential pool:** `/opt/data/auth.json` — 1304 B, mode 600. Structured pool with two anthropic credentials (indices [0] and [1]), each with id/label/auth_type/priority/source/access_token/base_url/last_status/last_error_*/request_count. access_token fields not read.

**Env vars in process:**
- ADMIN_PASSWORD (present in /proc/<pid>/environ — visible to same-uid processes)
- ADMIN_USERNAME (paired with above)
- No *_API_KEY / *_TOKEN / *_SECRET in environment.

**Not present:** /etc/* secrets, ~/.netrc, *.pem, *.key, keyring files.

## Open cleanup items

1. **Permissions on .env.bak files.** `chmod 600 /opt/data/.env.bak.*` or delete them. World-readable backup files leak the same key that the live .env protects.

2. **Compose-level vs runtime .env divergence.** Per earlier recon (handoffs/2026-05-07_hermes-openclaw-integration-findings.md side findings): `/docker/hermes-agent-bnz0/.env` holds the dead key, `/opt/data/.env` holds the live key. On `docker compose down && up` the dead key gets re-injected. Reconcile.

3. **ADMIN_PASSWORD as process env var.** Visible to anything running as the same uid via /proc. If a runtime-secret-fetch path exists, move it. Lower priority than items 1-2.

4. **Partial key fragment leaked to chat transcript during this recon.** The agent's awk parser split on `=` and surfaced a `sk-ant...DQAA` style prefix+suffix to the chat. Robert reviewed and elected not to rotate (deferred — has been rotating frequently and is choosing to absorb the partial-leak risk). Documented here for the audit trail.

## Discipline observations

The agent caught their own leak in real-time, stopped enumerating, named the bug class (`awk -F=` splitting on a value containing `=`), and surfaced rotation recommendation. They did not read auth.json access_token fields. They did not cat any credential file. Recon held discipline.

## Status

Items 1-3 are open. Item 4 is closed by Robert's decision (not rotating).

Next session can pick up items 1-3 by sending two commands to the Hostinger agent: `chmod 600 /opt/data/.env.bak.*` and a reconciliation of the two .env files.
