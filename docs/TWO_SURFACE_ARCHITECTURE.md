# Jayhawk Two-Surface Architecture (current)

## Surfaces
1) **Browser extension**
- Role: dispatch + tab automation.
- Produces: `runManifest` (schema pending in this repo).
- Storage: `chrome.storage.local` keys under `jayhawk.*`.

2) **Replit Sidecar/Jayhawk web UI**
- Role: compose mission, collect responses, synthesize Host Directive.
- Storage: localStorage keys: `jayhawk_huddles`, `jayhawk_custom_models`.
- Domain: Huddle + Dispatch; directive is the durable output.

## Rule
Contract-first, no integration implementation until `contracts/JAYHAWK_SHARED_CONTRACT_V0.md` is agreed.
