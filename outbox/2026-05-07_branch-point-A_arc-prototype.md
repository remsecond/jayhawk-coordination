---
title: "Branch Point A — Arc prototype paused for OFW corpus eval"
status: branch-point
type: marker
created: 2026-05-07
purpose: "Bookmark in the conversation. We were about to prototype an 'arc' as a block-tagged thread spanning multiple files. Roberto is grabbing the OFW corpus to evaluate the same idea against the higher-stakes legal data. Resume from this file when corpus eval is done."
---

# Branch Point A — Arc prototype paused for OFW corpus eval

## Where we left off

Roberto forwarded a Claude-session take on Obsidian blocks (now at `handoffs/2026-05-07_obsidian-blocks-take.md`). We engaged on the substance:

- File granularity loses internal structure.
- Block IDs (`^block-id`) + Block Properties + transclusion + MetadataCache make individual paragraphs first-class addressable, queryable objects.
- This solves the **"one issue, many threads" impracticality** — i.e., a conceptual arc that spans multiple files but doesn't belong to any single one.
- Arcs become first-class with: an arc-identity dimension, claim properties dimension, typed-relationships dimension. Three+ orthogonal dimensions, each independently queryable.

I named three arcs visible in this repo's current work:

1. `arc:credentials-migration` — 7 files: v0 §3, v0.1 §11.3.1–4, deployment notes, findings, cleanup next-steps, Telegram research, rehydration packet.
2. `arc:section-7-verification` — deployment notes §7, contract §7 (different §7!), rehydration packet §3, the bridge messages where the numbering collision cost us a round-trip.
3. `arc:allowlist-vs-double-lock` — findings doc, chat resolution, v0 §3 commentary, rehydration packet Finding 2 section.

Two candidate proof-of-concept arcs proposed:

- **(1) Our work** — `arc:credentials-migration`. Already in this repo, fully landed, recent. Cheap round-trip. Learn the Obsidian-block gotchas without case-stakes pressure.
- **(2) Roberto's case work** — `arc:financial-hardship` (or the highest-friction arc in the OFW corpus). Higher stakes; I don't have the corpus in-thread.

My recommendation was: start with (1), then apply lessons to (2). Roberto chose to fork: grab the OFW corpus first and evaluate against the higher-stakes data, then return.

## What's open when we resume

1. **Did the OFW corpus eval surface anything that changes the arc design?** Specifically: are there arcs in OFW that have richer relationship types (contradicts, supersedes, withdraws, contingent-on) than the ones in our coordination work? If yes, the typed-relationship vocabulary in the prototype should reflect them.
2. **Which corpus does the prototype run against — OFW or coordination?** If OFW eval reveals the value clearly, skip the coordination dry-run and prototype directly on the case data. Otherwise, fall back to the original recommendation: `arc:credentials-migration` first.
3. **Is the prototype run by the contract Claude (this session) or by another tool?** If the corpus is private case material, this Claude shouldn't see it; the prototype should run in a Claude session that's already trusted with the case data, with this session producing the design doc only.
4. **Tagspaces** — Roberto mentioned wanting to push on Tagspaces alongside Obsidian. Open thread; not yet engaged.

## Resume instructions

When ready to come back to arcs work:

1. Point this Claude session at `outbox/2026-05-07_branch-point-A_arc-prototype.md`.
2. Note any findings from the OFW corpus eval that should inform the prototype (richer relationship types, scale considerations, sensitivity boundaries).
3. Pick the prototype target: OFW (case data) or `arc:credentials-migration` (this repo).
4. If OFW: this session produces the design only; the prototype runs in a case-data-trusted session.
5. If `arc:credentials-migration`: this session can run the full prototype (retrofit block IDs into the 7 files, define the arc parent, link via typed relationships).

## Context pointers

- Forwarded Claude take on blocks: `handoffs/2026-05-07_obsidian-blocks-take.md`
- Forwarded Telegram research from same / similar session: `handoffs/2026-05-07_for-claw-telegram-research.md`
- Rehydration packet (full state of the Hermes/Claw work): `handoffs/2026-05-07_rehydration-packet.md`
- The seven files in `arc:credentials-migration`:
  - `contracts/SHARED_SURFACE_HOSTINGER_V0.md` §3
  - `contracts/SHARED_SURFACE_HOSTINGER_V0_1.md` §11.3.1–4
  - `docs/HOSTINGER_DEPLOYMENT_NOTES.md`
  - `handoffs/2026-05-07_hermes-openclaw-integration-findings.md` Finding 2
  - `handoffs/2026-05-07_hostinger-cleanup-next-steps.md` item 6
  - `handoffs/2026-05-07_for-claw-telegram-research.md` (dependency-chain explanation)
  - `handoffs/2026-05-07_rehydration-packet.md` §6, §8, Finding 2 section

---

*Branch point committed 2026-05-07. Resume by pointing this session at this file.*
