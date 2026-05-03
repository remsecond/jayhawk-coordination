# Extension runManifest extraction (PENDING)

Blocker: the extension-side verbatim extraction (buildRunManifest body or a redacted sample runManifest JSON) has not yet been provided in this workspace.

What we do have (from handoff notes):
- Contract-bearing hard-coded fields:
  - `next_operator: "Claude Tab"`
  - `requested_collection_mode: "poll_live_tabs"`

Next: paste either
- buildRunManifest function body (verbatim), or
- a single redacted sample `runManifest.json` from `chrome.storage.local` (jayhawk.runs.<runId>).
