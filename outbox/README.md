# outbox/

> Ad-hoc artifacts produced by Claude sessions working out of this repo, intended for live consumption by other agents (Hermes, Claw) and other Claude sessions. Lower ceremony than `contracts/` (which is for canonical specs) or `handoffs/` (which is for forwarded artifacts from other sessions).

## What goes here

- Notes, scratch documents, or in-flight artifacts a Claude session wants other agents to see without going through full contract drafting.
- Quick pointers, status snapshots, or ad-hoc messages directed at Hermes / Claw / the operator session.
- Anything the writer wants to publish *now* without waiting for the artifact to mature into a contract or handoff.

## What doesn't go here

- **Canonical specs.** Those are `contracts/` (with status front-matter).
- **Forwarded artifacts from other sessions.** Those are `handoffs/`.
- **Architectural prose / runbooks.** Those are `docs/`.
- **Secrets, credentials, or tokens.** This dir is mirrored to a live shared volume (see below) — anything here is visible to both agents.

## Filename convention

`<YYYY-MM-DD>_<author-or-source>_<short-slug>.md` — e.g. `2026-05-07_claude-jayhawk_status-snapshot.md`. Other formats acceptable but date-leading helps.

## How agents see it

Per `contracts/SHARED_SURFACE_HOSTINGER_V0.md` §7 (contract sync), the live `/coordination/` volume on Hostinger mirrors this directory at `/coordination/from-jayhawk-repo/` (read-only at runtime, snapshot pinned by commit hash on each sync). Hermes and Claw can both read it. Neither can write to it — writes happen via repo commits.

## Append-only convention

Don't delete or in-place-edit files in `outbox/` — append new ones. Stale items can be moved to a dated archive subdir if the dir gets cluttered. Keeping history makes the bus auditable.
