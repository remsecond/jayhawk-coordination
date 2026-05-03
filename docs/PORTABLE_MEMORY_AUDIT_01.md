# Portable Memory Audit 01 (Jayhawk / OBE)

Repo baseline: `remsecond/jayhawk-coordination` @ `8195470` (2026-05-03).

## 1. Executive finding

**Partially real, partially rhetorical.** The repo is functioning as a durable truth surface for *interfaces and evidence* (contract + verbatim snapshots). But the “Karpathy wiki / portable memory” pattern is **not yet fully implemented** because the repo lacks a small set of wiki-like primitives (Start Here, Current State, Decision Ledger) and some existing docs are stale, causing operator-carried context and drift.

## 2. Working definition of “Karpathy wiki” for Jayhawk / OBE

A lightweight, versioned, plaintext “wiki” where:
- new sessions can cold-start by reading **one entrypoint** and reach the current state quickly,
- **decisions and interfaces** are written down with links to evidence,
- updates are cheap and frequent (small commits),
- the system’s truth does not depend on chat scrollback or a person’s memory.

For Jayhawk/OBE, the wiki is the **slow layer**: this repo.

## 3. Evidence of actual implementation (verified)

- Canonical interface doc exists and is comprehensive: `contracts/JAYHAWK_SHARED_CONTRACT_V0.md` (includes verbatim schemas, mapping table, explicit mismatch handling, do-not-touch list, and open questions). (Path verified on `main`.)
- Verbatim evidence is stored as snapshots:
  - Replit App.tsx extraction: `replit-sidecar-snapshot/2026-05-02_replit-app-tsx-extraction.md`
  - Replit Q2/Q3 handler addendum: `replit-sidecar-snapshot/2026-05-03_replit-app-tsx-target-response-handler.md`
  - Extension runManifest extraction: `extension-snapshot/2026-05-02_extension-runmanifest-extraction.md`
  - Extension sample manifest: `extension-snapshot/extension.runManifest.sample.json`
- Durable integration artifact exists: `docs/INTEGRATION_PLAN_V0.md` (merged to `main` via PR). (Path verified on `main`.)
- Contract-bearing constraints are captured: `docs/DO_NOT_TOUCH.md`.

These are real portable-memory artifacts: a new actor can read them without the chat thread.

## 4. Evidence of gaps / false claims (verified)

- **Stale “current status”** in `README.md` claims the repo is waiting on verbatim Replit snippets and extension runManifest extraction/sample, but those artifacts are present in the repo (paths listed above). This is direct evidence that the slow layer is not being maintained as the “wiki”.
- `docs/TWO_SURFACE_ARCHITECTURE.md` says the extension “Produces: runManifest (schema pending in this repo)”, but the runManifest schema/extraction exists (`extension-snapshot/2026-05-02_extension-runmanifest-extraction.md`). Another drift signal.
- Missing wiki primitives:
  - No `docs/START_HERE.md` / `docs/CURRENT_STATE.md` equivalent.
  - No `docs/decisions/` directory or decision index/template.

Net: we have “contract + evidence”, but not the minimal navigation/state layer that makes it a Karpathy-style wiki for repeated cycles.

## 5. Current portable-memory artifacts (verified)

- Interface/contract: `contracts/JAYHAWK_SHARED_CONTRACT_V0.md`
- Integration plan (plan-only, no code): `docs/INTEGRATION_PLAN_V0.md`
- Constraints: `docs/DO_NOT_TOUCH.md`
- Architecture note: `docs/TWO_SURFACE_ARCHITECTURE.md` (currently stale in one line)
- Verbatim evidence:
  - `replit-sidecar-snapshot/*`
  - `extension-snapshot/*`
- Preserved context/handoffs:
  - `handoffs/2026-05-03_integration-readiness-assessment.md`
  - `replit-sidecar-snapshot/2026-05-03_replit-hardening-pass-report.txt`

## 6. Failure modes observed in Huddle 02 (inferred from repo + process)

- **Drift despite canonical repo** (verified by stale README/docs), implying actors or relay generated forward without re-reading repo state.
- **Auth friction and landing overhead**: presence of patch artifacts (e.g. `patches/huddle-02_integration-plan-v0.patch`) suggests the workflow relied on patch relay rather than direct PR automation.

## 7. Minimal fix (least new machinery)

Make the “wiki” real with three small additions and two small edits:

1) Add `docs/START_HERE.md` (entrypoint: what to read first, what is canonical, where the contract is).
2) Add `docs/CURRENT_STATE.md` (what’s true right now: what’s done, what’s next, what’s blocked).
3) Add `docs/decisions/DECISION_TEMPLATE.md` + `docs/decisions/INDEX.md` (thin decision ledger, linkable).
4) Update `README.md` “Current status” to match reality.
5) Update the one stale line in `docs/TWO_SURFACE_ARCHITECTURE.md` (“schema pending”).

No new platform, no automation required. Just navigation + state + decisions.

## 8. Recommendation

Treat “wiki upkeep” as a first-class deliverable of each huddle: end every huddle by landing either a decision entry or a CURRENT_STATE update (small commit). Then run Huddle 03 with a single artifact target: land the three primitives above and bring README/docs back into sync.
