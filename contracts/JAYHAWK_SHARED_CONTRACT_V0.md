---
title: Jayhawk Shared Contract v0
status: candidate
type: spec
spec_version: v0
owner: roberto
authors:
  - dispatch (claude)
created: 2026-05-02
last_updated: 2026-05-03
revision_notes:
  - 2026-05-02 initial draft (Replit Huddle schema partial / inferred)
  - 2026-05-02 Replit App.tsx schema verified verbatim; [unconfirmed] tags removed; mismatches in §3 §4 §5 §6 made explicit; Q1 closed in §10
  - 2026-05-03 Replit MAX_HUDDLES + per-target response handler extracted; Q2/Q3 closed in §10
covers:
  - C:\Users\ClawDaddy\Documents\New project 2 (extension source)
  - C:\Users\ClawDaddy\Documents\jayhawk-v0.1.0 (extension installed)
  - https://jayhawk-sidecar-control.replit.app (Replit Sidecar app)
related:
  - 10_Platforms/Sidecar/02_Specs/JAYHAWK_DISPATCH_UI_ARCHITECTURE.md
  - 10_Platforms/Sidecar/02_Specs/PORTABLE_MEMORY_CORE_V0.md
  - extension-snapshot/2026-05-02_extension-runmanifest-extraction.md (in coordination repo)
  - replit-sidecar-snapshot/2026-05-02_replit-app-tsx-extraction.md (in coordination repo)
  - replit-sidecar-snapshot/2026-05-03_replit-app-tsx-target-response-handler.md (in coordination repo)
note: "Documentation only. No code changes. No renames. No Card / Obsidian / provenance integration in v0."
---

# Jayhawk Shared Contract v0

> **Status: candidate.** Documentation only. This spec defines the minimal shared contract between two existing Jayhawk implementations so they can interoperate without either rewriting the other. Adoption requires a separate Roberto-promoted directive.
>
> **Revision 2026-05-02:** Replit App.tsx schema is now verified verbatim (see `replit-sidecar-snapshot/2026-05-02_replit-app-tsx-extraction.md`). All `[unconfirmed]` tags removed. Real schema mismatches surfaced in §3, §4, §5, §6.
>
> **Revision 2026-05-03:** Replit `MAX_HUDDLES` value and the per-target response handler (`setTargetResponse`) are now verified (see `replit-sidecar-snapshot/2026-05-03_replit-app-tsx-target-response-handler.md`).

---

## 1. Purpose

Two Jayhawk implementations exist today:

- **Replit Sidecar app** (`@workspace/sidecar`, deployed at `jayhawk-sidecar-control.replit.app`) — composition surface. Mission Composer UX, modes/targets, preview, Active and Recent Huddles, Host Directive synthesis, "Promote to Obsidian — later" placeholder. Storage: `localStorage`. No automation; user manually pastes responses.
- **Local Chrome extension** (source: `New project 2`; deployed: `jayhawk-v0.1.0`) — dispatch surface. Popup that yields after Go (Phase 1, 2026-05-02), side panel, content-script auto-paste into ChatGPT/Claude/DeepSeek/Gemini, run-manifest archive. Storage: `chrome.storage.sync` + `chrome.storage.local`.

These do not compete. They implement complementary layers of one workflow:

> **Replit composes. Extension dispatches. The bridge is a contract, not shared code.**

The contract below names the load-bearing fields each side already produces, proposes a unified shape neither side has to fully adopt yet, and surfaces the lifecycle/storage/naming differences that need a deliberate decision before any handoff is wired.

This v0 is documentation. No code is changed. No keys are renamed. No Card/Obsidian/provenance machinery is introduced.

---

## 2. Current Extension runManifest schema (verified verbatim from `background.js`)

Built by `buildRunManifest({runId, prompt, sites, opened, results, groupId})` and stored under `jayhawk.activeRun` plus `jayhawk.runs.<runId>` in `chrome.storage.local`.

```js
{
  run_id: "jh_<YYYYMMDDHHMMSS>_<6-char-rand>",
  prompt_hash: "sha256:<lowercase-hex>",
  prompt_title: <prompt whitespace-collapsed, sliced to 80 chars>,
  targets: [<site display name or url, in order>],
  target_tab_urls: [<live chrome.tabs.get(tabId).url, fallback to site.url, fallback to "">],
  pasted_count: <count of results where r.ok === true>,
  created_at: <ISO 8601 timestamp>,
  next_operator: "Claude Tab",                        // hard-coded literal
  requested_collection_mode: "poll_live_tabs",        // hard-coded literal
  group_id: <chrome tab-group id> | null,
  target_tab_ids: [<integer chrome tab ids>]
}
```

Per-result element shape (members of a transient `results` array; not stored in manifest verbatim, but counted into `pasted_count`):

```js
{
  name: <site display name or url>,
  tabId: <chrome tab id>,                              // present when paste was attempted
  ok: <boolean>,
  submitted: <boolean>,
  reason: <string>                                     // present on failure paths
                                                        // one of: "injector-missing" | "no-result" |
                                                        // "tab-error" | "load-timeout" | other
                                                        // (extras spread from __jayhawkPaste return)
}
```

Site object after `sanitizeSites` in background:

```js
{ name: <string>, url: <normalized https url> }
// `enabled` is stripped before the manifest layer
```

Full extraction (function bodies, message envelope, storage keys) lives in `extension-snapshot/2026-05-02_extension-runmanifest-extraction.md` in the coordination repo.

---

## 3. Current Replit Huddle schema (verified verbatim from `App.tsx`)

Full type body and lifecycle code extracted 2026-05-02. Source artifact: `replit-sidecar-snapshot/2026-05-02_replit-app-tsx-extraction.md`.

### `Huddle` type (lines 15–27)

```ts
type Huddle = {
  id: string;
  status: HuddleStatus;
  compiledPrompt: string;          // the FULL compiled prompt text (with the "# Jayhawk dispatch" wrapper)
  outcomeSnapshot: string;         // mission-goal field from the composer
  instructionsSnapshot: string;    // discussion/instructions field from the composer
  modesSnapshot: string[];         // preset LABELS for display
  presetIdsSnapshot: string[];     // preset IDS for re-rendering
  targets: Target[];
  host: HostChoice;                // who synthesizes the directive (NOT who collects responses — see §6)
  hostDirective: string;
  createdAt: number;               // epoch milliseconds via Date.now() — NOT ISO string
};
```

### Supporting types

```ts
type HuddleStatus  = "active" | "complete" | "abandoned";
type DispatchStatus = "idle" | "preparing" | "opening" | "sent" | "failed";  // UI-only, not stored
type Model         = { id: string; label: string; url?: string; custom?: boolean };
type Preset        = { id: string; label: string; prompt: string };
type TargetStatus  = "pending" | "opened" | "failed" | "complete";
type HostChoice    = "GPT" | "Claude";
type Target        = { modelId: string; label: string; status: TargetStatus; response: string };
```

### Storage keys (verified)

```ts
const HUDDLES_KEY       = "jayhawk_huddles";        // Huddle[] in localStorage, capped by MAX_HUDDLES
const CUSTOM_MODELS_KEY = "jayhawk_custom_models";  // user-defined Model[] additions
const THEME_KEY         = "jayhawk_theme";          // ThemeSetting (UI only — not contract-relevant)
```

### Compiled prompt structure (verified — emitted by lines 403–447 IIFE)

```
# Jayhawk dispatch
You are one of N models receiving this prompt in parallel.
Targets: ChatGPT, DeepSeek, Gemini

Modes: <preset labels comma-separated>     (only if presets selected)

## Outcome                                  (only if outcome present)
<outcome text trimmed>

## Instructions                             (only if instructions present)
<instructions text trimmed>

---
Respond directly. Do not coordinate with other models. Do not assume shared context.

<!-- dispatch: close-previous-tabs=true -->  (only if flag on)
```

### Lifecycle (verified)

| Transition | Trigger |
|---|---|
| (none) → `active` | `handleGo()` creates new Huddle |
| `active` → `complete` | `copyDirective()` succeeds (clipboard write OK) |
| `active` → `abandoned` | `reconcilePriorActive(newId)` called by next `handleGo` for the previous active Huddle |
| Per-target `(n/a)` → `opened` | `handleGo()` initializes all targets as `opened` (skipping `pending` to avoid UI flicker) |
| Per-target `opened` → `complete` | `setTargetResponse(modelId, text)` sets `Target.status = "complete"` when `text.trim().length >= 2` |
| Per-target `opened` → `failed` | Not observed in extracted snippets (enum includes `failed`, but no setter found in extracted handlers) |

Invariant: at most ONE Huddle has `status: "active"` at a time.

`DispatchStatus` is UI-only and never persisted. It drives a transient animation: `preparing → opening → sent → idle` over ~4.4 seconds.

---

## 4. Mapping table

| Concept | Extension field | Replit field | Same shape? | Notes |
|---|---|---|---|---|
| Run / Huddle identifier | `run_id` (`"jh_<ts>_<rand>"`) | `id` (`uid()` — format not in extraction) | both string; format differs | |
| Prompt body (compiled) | (none — only `prompt_hash` + `prompt_title` stored) | `compiledPrompt: string` (full text) | mismatch — fidelity gap | Replit stores body; extension drops it |
| Prompt body (raw input) | (transmitted on `FANOUT`, never stored) | `outcomeSnapshot` + `instructionsSnapshot` (two fields) | mismatch | Replit splits outcome from instructions; extension is one blob |
| Prompt hash | `prompt_hash: "sha256:<hex>"` | (none — could be derived from `compiledPrompt`) | extension-only | |
| Prompt title | `prompt_title: <80 chars>` | (could be derived from `outcomeSnapshot`) | extension-only | |
| Outcome / mission | (none) | `outcomeSnapshot: string` | Replit-only | |
| Instructions | (none) | `instructionsSnapshot: string` | Replit-only | |
| Modes / presets | (none) | `modesSnapshot: string[]` (labels) AND `presetIdsSnapshot: string[]` (ids) | Replit-only; TWO arrays | Both stored intentionally — labels for display, ids for re-render |
| Targets / participants | `targets: string[]` (display names) AND `target_tab_urls: string[]` (parallel array) | `targets: Target[]` (`{modelId, label, status, response}` objects) | mismatch — both lists; element shape differs | Extension parallel arrays; Replit object array |
| Per-target status | `results[].ok` + `results[].submitted` + `results[].reason?` (booleans + string) | `Target.status: TargetStatus` (`"pending" \| "opened" \| "failed" \| "complete"`) | mismatch | Replit has richer enum; extension has booleans + reason |
| Per-target response text | (none — extension does not collect responses) | `Target.response: string` (verbatim user-pasted text) | Replit-only | Extension's collection boundary stops at "did paste succeed"; Replit captures what the model said |
| Tab IDs | `target_tab_ids: number[]` | (none) | extension-only | No browser context in Replit |
| Tab live URLs | `target_tab_urls: string[]` | (none) | extension-only | |
| Tab group | `group_id: number \| null` | (none) | extension-only | |
| Pasted count | `pasted_count: number` | (derivable from `targets[].status === "complete"`) | extension stores; Replit derives | |
| Status (whole-object) | (none — manifest is post-hoc) | `status: HuddleStatus` | mismatch | Extension manifest is terminal; Replit Huddle has lifecycle |
| Created timestamp | `created_at: <ISO 8601 string>` | `createdAt: number` (epoch ms) | **TYPE MISMATCH** | See §6.1 |
| Completed timestamp | (none) | (not in extracted type — Huddle moves to `complete` but no completedAt field stored) | both incomplete | |
| Synthesizer | (none) | `host: HostChoice` (`"GPT" \| "Claude"`) | Replit-only | Who writes the directive |
| Host directive | (none) | `hostDirective: string` | Replit-only | The synthesized output |
| Collector / next operator | `next_operator: "Claude Tab"` (hard-coded) | (none) | extension-only | Who gathers responses; see §6.3 — DIFFERENT from `host` |
| Collection mode | `requested_collection_mode: "poll_live_tabs"` (hard-coded) | (implicit — manual paste into Huddle UI) | extension-only declaration | |

**Reading of the table:** the extension produces a *post-dispatch artifact* (manifest); Replit produces a *full lifecycle record* (Huddle with stateful transitions and verbatim response text). They overlap on identity (id, prompt-hash-able content, target list) but diverge sharply on (a) what's stored about the prompt, (b) per-target richness, (c) timestamp type, and (d) the synthesizer-vs-collector role distinction.

---

## 5. Proposed unified contract

A single shape that supersets both sides without renaming either's storage today. Either side can adopt incrementally; the contract is a target, not a forced migration.

```yaml
# Jayhawk shared contract v0 (proposed)
schema_version: "1"
contract_id: jayhawk.shared.v0

object_kind: jayhawk_huddle    # one type, not two
                                # subsumes "Huddle" (Replit) and "runManifest" (extension)

# --- Identity ---
id: string                      # canonical: "jh_<YYYYMMDDHHMMSS>_<6-char-rand>"
                                # extension format; Replit's uid() format adopts this if/when they
                                # converge

# --- Prompt (all three forms; nullable per-side) ---
prompt:
  compiled: string | null       # full compiled prompt with "# Jayhawk dispatch" wrapper
                                # Replit always populates; extension may not store (currently doesn't)
  outcome: string | null        # mission goal — Replit-only today
  instructions: string | null   # discussion/instructions — Replit-only today
  hash: string                  # "sha256:<lowercase-hex>" — derivable from compiled if present
  title: string                 # first 80 chars whitespace-collapsed of compiled or outcome

# --- Composition modes (Replit-side) ---
composition:
  mode_labels: string[] | null  # display labels (Replit's modesSnapshot)
  mode_ids: string[] | null     # stable ids (Replit's presetIdsSnapshot)

# --- Targets / participants ---
participants:
  - id: string                  # stable per-target id within this huddle (Replit's modelId)
    name: string                # display name (Replit's label, extension's targets[i])
    url: string | null          # normalized https URL of the model surface
    # extension-only enrichment (nullable when authored from Replit):
    tab_id: number | null
    tab_url: string | null      # live url after navigation
    group_id: number | null

# --- Execution / responses ---
responses:
  - participant_id: string      # FK to participants[].id
    # extension view (paste-state):
    ok: boolean | null          # extension-only — did paste succeed
    submitted: boolean | null   # extension-only — did submit fire
    paste_reason: string | null # extension-only failure reason
    # Replit view (per-target lifecycle + content):
    status: string | null       # one of "pending" | "opened" | "failed" | "complete"
    response_text: string | null  # Replit-only; verbatim user-pasted model response
    received_at: string | null    # ISO 8601; null until populated

# --- Lifecycle ---
status: "draft" | "dispatched" | "collecting" | "complete" | "abandoned"
                                # superset of Replit's HuddleStatus + extension's implicit lifecycle
created_at: string              # ISO 8601 — chosen format (see §6.1 for type-mismatch resolution)
dispatched_at: string | null    # ISO 8601 — when extension's runFanout finished
completed_at: string | null     # ISO 8601 — when host directive was finalized

# --- Synthesis (Replit-side) ---
synthesizer:                    # the role-name "synthesizer" disambiguates from "collector" — see §6.3
  agent: string | null          # who synthesizes — currently "GPT" or "Claude" per Replit's HostChoice
  directive_text: string | null # Replit's hostDirective
  directive_status: "draft" | "promoted" | "discarded" | null

# --- Handoff contract (the integration "next-step" declaration) ---
collector:                      # the role-name "collector" disambiguates from "synthesizer" — see §6.3
  agent: string                 # currently "Claude Tab" (extension's hard-coded next_operator)
  collection_mode: string       # currently "poll_live_tabs" (extension's hard-coded requested_collection_mode)
```

This shape is a **superset** — neither implementation has to populate every field today. The contract describes what the field MEANS when it exists, not that it must exist.

---

## 6. Explicit treatment of the load-bearing schema mismatches

### 6.1 `createdAt: number` vs `created_at: <ISO string>` — TYPE MISMATCH

- **Status today:** Replit uses `Date.now()` (epoch milliseconds); Extension uses `new Date().toISOString()` (ISO 8601 string).
- **Contract treatment:** unified contract uses **ISO 8601 string** as canonical. Rationale: self-describing, timezone-explicit, sortable as string, JSON-stable across language ecosystems.
- **Migration cost:** trivial — `new Date(createdAt).toISOString()` (Replit-side) and `Date.parse(created_at)` (any consumer needing ms).
- **What each side does today:** **NOTHING CHANGES.** Both sides keep producing what they produce. Any consumer reading the unified contract converts on read. v0.1 may unify the storage representation; v0 just names the canonical format.

### 6.2 Prompt fidelity mismatch (`compiledPrompt` vs `prompt_hash`)

- **Status today:** Replit stores the full compiled prompt as a string on the Huddle. Extension stores only a SHA-256 hash + an 80-char title; the full body is gone after dispatch.
- **Contract treatment:** the unified contract carries `prompt.compiled: string | null` AND `prompt.hash: string` (always). Replit populates both. Extension populates only `hash` and `title`.
- **Why this matters:** any cross-surface handoff that requires re-rendering the prompt (e.g. extension manifest → Replit Active Huddle UI) must come from the Replit side, because the extension cannot reconstruct the body. The contract makes this asymmetry explicit rather than papering over it.
- **What each side does today:** **NOTHING CHANGES.** Replit keeps storing the full body. Extension keeps storing only the hash. v0.1 may have the extension start storing the body too if the cost is acceptable; v0 names the gap.

### 6.3 `host` vs `next_operator` — DIFFERENT FIELDS, DIFFERENT ROLES

This is the most important clarification in the v0 revision.

- **Replit `host: HostChoice` (`"GPT" \| "Claude"`)** = the **synthesizer**. The agent that writes the host directive AFTER responses arrive. Set when the user picks who they want to synthesize.
- **Extension `next_operator: "Claude Tab"` (hard-coded)** = the **collector**. The agent that gathers model responses BEFORE synthesis. Hard-coded today because Tab is the only collector currently in the room.

These are TWO DIFFERENT LIFECYCLE ROLES:

```
dispatch → collect → synthesize → directive
            ^^^^^      ^^^^^^^^^
         collector   synthesizer
        (extension's    (Replit's
        next_operator)   host)
```

- **Contract treatment:** the unified contract names them separately: `collector.agent` and `synthesizer.agent`. Renaming both to role-clear names avoids the ongoing risk of either side misinterpreting the other's field.
- **Migration:** Replit can leave `host` as a synonym pointing at `synthesizer.agent`. Extension can leave `next_operator` as a synonym pointing at `collector.agent`. Neither field is renamed in code.
- **What this surfaces:** the extension currently DOES NOT name a synthesizer (it has no opinion on who writes the directive — that's downstream of collection). The Replit Huddle currently DOES NOT name a collector (it implicitly assumes "the user" via manual paste). The unified contract makes both fields visible; either side can leave its non-owned field null.

### 6.4 `requested_collection_mode: "poll_live_tabs"` (preserved)

- **Status today:** hard-coded literal in extension. Declares "the collector should poll the live tabs at `target_tab_urls` to gather responses."
- **Contract treatment:** preserved as `collector.collection_mode` default. Future modes (`"manual_paste_into_huddle"`, `"webhook_per_response"`, etc.) are theoretical until needed.
- **Replit behavior today:** implicit `"manual_paste_into_huddle"` — user pastes responses into Huddle target rows. v0 does not require Replit to set this field.

---

## 7. Storage key notes

The two implementations use **different namespacing conventions** for storage keys. **Do not rename either set in v0.** Renaming would lose existing user state on whichever side gets renamed.

| Side | Convention | Examples |
|---|---|---|
| Extension | dot-namespaced (`jayhawk.<noun>` and `jayhawk.<noun>.<id>`) | `jayhawk.sites`, `jayhawk.runs.<runId>` |
| Replit | underscore-namespaced (`jayhawk_<noun>`) | `jayhawk_huddles`, `jayhawk_custom_models`, `jayhawk_theme` |

### v0 policy

- Both conventions stay as they are.
- The contract document references both; neither codebase needs to rewrite storage code.
- Future v0.1 may converge if migration cost is acceptable — out of scope here.

### Drift risk to flag now

- The extension stores per-run manifests under `jayhawk.runs.<runId>` (one key per run). Replit stores all Huddles in a single `jayhawk_huddles` array (capped by `MAX_HUDDLES = 12`). Even with schema convergence, reading one from the other requires a transform.
- The extension uses both `chrome.storage.sync` (UI prefs) and `chrome.storage.local` (run manifests). Replit uses only browser `localStorage`. Cross-device behavior differs — extension prefs sync across the user's Chrome, Replit Huddles do not.
- These are storage-layer realities, not contract concerns. The contract is about object shape, not where the object lives.

---

## 8. Lifecycle mismatch notes

### Extension run lifecycle (recap)

The extension does not have an explicit lifecycle state. The implicit phases:

1. User opens popup or side panel → composes prompt + selects sites.
2. User clicks Go → `FANOUT` message fires, `runFanout` begins.
3. Background opens tabs, groups them, injects content script, attempts paste.
4. `buildRunManifest` runs at the END of `runFanout`. The manifest is a **post-hoc record**, not a live state object.
5. Manifest stored at `jayhawk.activeRun` and `jayhawk.runs.<runId>`. No further state transitions.
6. Per-tab progress is broadcast as `FANOUT_PROGRESS` events to the source tab during the run; collection is **not the extension's job** — see `next_operator: "Claude Tab"`.

**No `status` field.** Every manifest is terminal.

### Replit Huddle lifecycle (verified)

| Transition | Trigger | Code |
|---|---|---|
| (none) → `active` | new Huddle created | `handleGo()` |
| `active` → `complete` | host directive successfully copied to clipboard | `copyDirective()` |
| `active` → `abandoned` | next `handleGo()` runs `reconcilePriorActive(newId)` on the previous active Huddle | `reconcilePriorActive` |
| per-target `(n/a)` → `opened` | new Huddle initializes all targets as `opened` (skipping `pending` to avoid UI flicker) | `handleGo()` |
| per-target `opened` → `complete`/`failed` | code path not in extracted snippets | (presumed user paste / error) |

**Invariant:** at most ONE Huddle has `status: "active"` at a time.

### Directive completion behavior (verified)

The Huddle moves `active → complete` ONLY when `copyDirective()` successfully writes to the clipboard. There is NO separate "save directive" step. **The copy IS the completion signal.** If the clipboard write fails, the Huddle stays `active`.

### v0 contract treatment

The unified `status` enum:

| Status | Set by | Meaning |
|---|---|---|
| `draft` | Replit (composition in progress) | extension never sets this — it has no draft state |
| `dispatched` | extension (manifest just written) | when extension produces a manifest, it's already past dispatch |
| `collecting` | (future) Tab polling, or Replit waiting on user paste | not currently set by either side |
| `complete` | Replit (`copyDirective` succeeded) | extension never sets this — it doesn't track completion |
| `abandoned` | Replit (`reconcilePriorActive`) | extension has no equivalent |

**Important:** v0 does NOT require either side to start setting `status` it doesn't currently track. The contract just names the union. Adopters fill what they have and leave the rest null.

---

## 9. Do-not-touch list (v0)

Reaffirms boundaries from the integration readiness assessment:

- **Extension `runManifest` field names and values** (verified verbatim in §2). Renames break existing consumers including `jayhawk.runs.<runId>` archive.
- **Extension storage keys** (`jayhawk.sites`, `jayhawk.lastPrompt`, `jayhawk.closePrev`, `jayhawk.hideAfterGo`, `jayhawk.lastGroup`, `jayhawk.activeRun`, `jayhawk.runs.<runId>`). Existing user state.
- **Extension hard-coded literals** `"Claude Tab"` and `"poll_live_tabs"`. Treatment in §6; preserve as defaults.
- **Extension message types** (`FANOUT`, `FANOUT_PROGRESS`, `FANOUT_DONE`, `OPEN_SIDE_PANEL`, `JAYHAWK_OPEN_DRAWER`). Three sender surfaces wired against these strings.
- **Extension `content-injector.js` SELECTORS map.** Whatever it says today is what works against current ChatGPT/Claude/DeepSeek/Gemini DOMs.
- **Replit `Huddle` type** as currently defined. Contract may propose additions; do NOT delete or rename verified fields.
- **Replit storage keys** (`jayhawk_huddles`, `jayhawk_custom_models`, `jayhawk_theme`). Existing user state.
- **Replit `createdAt: number` representation.** Per §6.1, the contract uses ISO string canonically; Replit's storage stays epoch ms; conversion happens on read. Do NOT change Replit's storage shape in v0.
- **Replit "Promote to Obsidian — later" placeholder.** Out of scope for v0.
- **Cards / Anchors / Provenance / Obsidian integration.** Out of scope per directive.
- **Both Cipher Brain Export tooling and Room Command Bar v0** in "New project 2" — separate projects.
- **Production Replit deployment.** Live; do not modify during contract design.

---

## 10. Open questions

Recorded as open, not blockers. v0 is documentation; these can be resolved in v0.1 or later.

1. ~~**Full Replit Huddle type body.**~~ **RESOLVED 2026-05-02** via App.tsx extraction. See `replit-sidecar-snapshot/2026-05-02_replit-app-tsx-extraction.md`.
2. ~~**`MAX_HUDDLES` value** in Replit's `saveHuddles`.~~ **RESOLVED 2026-05-03:** `MAX_HUDDLES = 12`. See `replit-sidecar-snapshot/2026-05-03_replit-app-tsx-target-response-handler.md`.
3. ~~**Per-target completion code path** in Replit (`opened → complete` / `opened → failed`).~~ **RESOLVED (partial) 2026-05-03:** `setTargetResponse(modelId, text)` sets `Target.response` and flips `Target.status` to `"complete"` when `text.trim().length >= 2` (and can revert `complete → opened` if text becomes non-meaningful). A `failed` setter was not observed in the extracted handler. See `replit-sidecar-snapshot/2026-05-03_replit-app-tsx-target-response-handler.md`.
4. **Storage namespace convergence (`jayhawk.` vs `jayhawk_`).** v0 keeps both. v0.1 picks one if migration cost is acceptable.
5. **`next_operator` and `requested_collection_mode` as enums vs literals.** Today both are hard-coded. v0 preserves; v0.1 may enumerate when alternatives appear.
6. **Lifecycle states extension doesn't track.** Should the extension start setting `dispatched`/`collecting`? Or stay terminal-manifest-only? v0 allows both; v0.1 may pick.
7. **Cross-surface handoff transport.** How does an extension manifest reach Replit's Active Huddle? Manual paste is the v0 answer; the proposed `_ai/interop/` is a v0.1+ candidate; the `jayhawk-coordination` repo itself may serve as a canonical handoff surface.
8. **Versioning policy.** `schema_version: "1"`. v0 commits to bumping on breaking change; validation tooling deferred.
9. **`participant_id` stability.** §5 introduces a `participant_id` for cross-referencing responses. Replit has `Target.modelId` (stable across the Huddle's lifetime); extension uses position. Adoption requires extension to mint stable ids per dispatch.
10. **Contract location and ownership.** The `jayhawk-coordination` git repo (Claw, 2026-05-02) is the canonical home — both codebases reference it; neither owns it. v0.1 may add a `contract.ts` / `contract.js` import on each side that mirrors the schema literally.
11. **Dual storage of `modesSnapshot` + `presetIdsSnapshot`.** Replit stores both labels and ids intentionally. The unified contract preserves both as `mode_labels` + `mode_ids`. Confirm this is the right design choice in v0.1.

---

*End of Jayhawk Shared Contract v0 (revised 2026-05-03). Candidate. Documentation only. Promotion to canonical requires Roberto sign-off after the open questions in §10 are at least triaged. No code changes follow from filing this document.*
