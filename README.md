# Jayhawk Coordination Repo (Public)

This repo is the shared coordination surface for Jayhawk.

## Goal
Provide a single cloud-accessible place to share and review Jayhawk artifacts **without** copy/paste relay.

## Scope (current)
- Contract-first only (no integration implementation yet).
- Preserve two separate surfaces:
  - Browser extension (dispatcher + runManifest producer)
  - Replit Sidecar/Jayhawk app (Huddle composer + directive sink)

## Repo layout
- `contracts/` — shared contract docs (canonical)
- `replit-sidecar-snapshot/` — schema/type/function snapshots extracted from Replit Sidecar
- `extension-snapshot/` — schema/type/function snapshots extracted from extension source
- `docs/` — architecture notes, operating rules
- `handoffs/` — pasted handoffs/reports that should be preserved verbatim

## Operating rule
Chat is not a source of truth. If it matters, it lands here as a file + commit.

## Current status
- Waiting on verbatim Replit snippets from `artifacts/sidecar/src/App.tsx`.
- Waiting on verbatim extension `runManifest` extraction/sample.

