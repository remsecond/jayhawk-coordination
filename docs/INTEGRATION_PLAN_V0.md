# Jayhawk Integration Plan — V0

Status: candidate plan. Lands when this file is on main.
Source of truth: contracts/JAYHAWK_SHARED_CONTRACT_V0.md.
This is a plan, not code.

## 1. Purpose

Define the minimum surface for a Replit Huddle and an Extension runManifest to round-trip through JayhawkSharedV0 JSON via manual paste, without further operator decisions. Deeper scope is deferred.

## 2. Existing contract reference

All schemas, the mapping table, the four known mismatches, the do-not-touch list, and the eleven open questions live in contracts/JAYHAWK_SHARED_CONTRACT_V0.md. This plan does not restate them and does not force edits to that file.

## 3. Replit adapter target

- Path: artifacts/sidecar/src/interop/contract.ts.
- Exports: toContract(h: Huddle): JayhawkSharedV0, fromContract(x: JayhawkSharedV0): Partial<Huddle>.
- Behavior: warn-only; no throws on the four known mismatches.
- Owner: Code writes; Roberto lands.

## 4. Extension adapter target

- Path: src/interop/contract.js or lib/interop/contract.js (final path picked at write time).
- Exports: toContract(runManifest): JayhawkSharedV0, fromContract(x: JayhawkSharedV0): { prompt, sites }.
- Behavior: warn-only; if stable participant_id cannot be minted in v0, leave null and log.
- Owner: Code writes; Roberto lands.

## 5. Manual paste round-trip

Three observable steps, no automated transport:

1. In Replit devtools, run toContract(activeHuddle); copy JSON.
2. Paste JSON into the extension import surface (textarea or devtools); extension validates shape, logs warns.
3. In the extension, run toContract(runManifest); copy JSON; paste into Replit import surface; Replit renders read-only Huddle preview via fromContract.

Import surfaces may be devtools console or a temporary textarea. UI choice is implementer's call.

## 6. Validation gates

Both adapters validate shape only. They do not codify the four known mismatches; the contract doc remains authoritative. A diff'd round-trip is acceptable if the diff is recorded in the run log named in Section 7.

## 7. Definition of done

This huddle lands when docs/INTEGRATION_PLAN_V0.md is on main. That is the only landing requirement for Huddle 02.

Execution follow-ups, tracked separately and not gating this huddle:

- Adapter stubs committed at the targets in Sections 3 and 4 (signatures + null-population only).
- One round-trip log committed at runs/2026-05-03_huddle-02_round-trip-test.md, showing a clean round-trip or a documented diff.

## 8. Deferred

- Transport beyond manual paste (clipboard helper, URL scheme, GitHub-as-bus, webhook) — Huddle 03.
- Replit "Dispatch via extension" UI; extension auto-pickup of Replit manifests; cross-repo code consolidation.
- Open contract questions that stay open: #4, #5, #6, #8, #9, #10.10, #11.
- Closed by this plan: #7 (manual paste as v0 transport), #10 (per-side contract.{ts,js} as v0.1 candidate).

## 9. Risks

- Stub creep: adapters expand into full mapping mid-flight. Mitigation: null-population only; full mapping is a later huddle.
- Transport drift: implementer adds a non-paste transport. Mitigation: Section 8 is binding; reviewer rejects.
- Unverified push paths for extension and Replit repos. Mitigation: Code confirms at first push; treat as fast-fix, not re-plan.
- Process overfitting: huddle protocol over-shapes outputs. Mitigation: protocol stays advisory after Huddle 02.
