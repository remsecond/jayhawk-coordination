# JAYHAWK_SHARED_CONTRACT_V0

Status: DRAFT (contract-first, no integration yet)

## 0. Purpose
Define a shared contract between:
- Jayhawk browser extension (dispatcher, runManifest producer)
- Replit Sidecar/Jayhawk web UI (Huddle composer, directive sink)

## 1. Non-goals
- No integration implementation here.
- No renames of existing keys/message types/selectors.
- No architecture changes.

## 2. Canonical objects (verbatim)
### 2.1 Extension: runManifest (VERBATIM)
See: `extension-snapshot/2026-05-02_extension-runmanifest-extraction.md`

Hard-coded contract-bearing fields (must be acknowledged):
- `next_operator: "Claude Tab"`
- `requested_collection_mode: "poll_live_tabs"`

### 2.2 Replit: Huddle (VERBATIM)
> Paste verbatim types + related helpers here.

Required snippets (from `artifacts/sidecar/src/App.tsx`):
1) Huddle type
2) HuddleStatus
3) DispatchStatus
4) participant/model/target types
5) prompt compiler function (contains `lines.push("# Jayhawk dispatch")`)
6) Huddle creation function
7) setHostDirective
8) copyDirective
9) loadHuddles / saveHuddles
10) localStorage key constants

## 3. Mapping table
| Concept | Extension field(s) | Replit field(s) | Notes | Blocking? |
|---|---|---|---|---|
| run identity |  |  |  |  |
| targets/participants |  |  |  |  |
| outcome |  |  |  |  |
| instructions |  |  |  |  |
| compiled prompt |  |  |  |  |
| collection mode | requested_collection_mode |  |  |  |
| next operator | next_operator |  |  |  |
| host directive |  | hostDirective |  |  |
| persistence keys | chrome.storage keys | localStorage keys | do-not-touch |  |

## 4. Do-not-touch risks
See: `docs/DO_NOT_TOUCH.md`

## 5. Open questions (must be answered explicitly)
- Does Replit acknowledge `next_operator="Claude Tab"` as-is?
- Does Replit acknowledge `requested_collection_mode="poll_live_tabs"` as-is?
- If not, what is the minimal compatible translation layer (doc-only decision, no code yet)?

## 6. Suggested edits log
> When reviewers suggest edits, add them here with dates.

