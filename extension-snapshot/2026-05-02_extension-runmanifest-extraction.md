# Extension runManifest extraction — verbatim

**Source:** `C:\Users\ClawDaddy\Documents\New project 2\background.js` (extension source-of-truth)
**Extracted:** 2026-05-02 by Dispatch via read-only file inspection
**Purpose:** fill `extension-snapshot/EXTENSION_RUNMANIFEST_EXTRACTION_PENDING.md` placeholder in `jayhawk-coordination` repo
**Read-only.** No code changes; this is a snapshot for the contract-first integration plan.

---

## 1. `buildRunManifest` (full function, verbatim)

Local helpers used: `api(chrome.tabs, "get", ...)` (defined at the bottom of `background.js`) and `hashPrompt(prompt)` (defined immediately below). Context (the next function `hashPrompt`) included.

```js
async function buildRunManifest({ runId, prompt, sites, opened, results, groupId }) {
  const target_tab_urls = await Promise.all(opened.map(async item => {
    if (!item.tabId) return item.url || "";
    try {
      const tab = await api(chrome.tabs, "get", item.tabId);
      return tab.url || item.url || "";
    } catch {
      return item.url || "";
    }
  }));
  return {
    run_id: runId,
    prompt_hash: await hashPrompt(prompt),
    prompt_title: prompt.replace(/\s+/g, " ").trim().slice(0, 80),
    targets: sites.map(site => site.name || site.url),
    target_tab_urls,
    pasted_count: results.filter(result => result.ok).length,
    created_at: new Date().toISOString(),
    next_operator: "Claude Tab",
    requested_collection_mode: "poll_live_tabs",
    group_id: groupId,
    target_tab_ids: opened.map(item => item.tabId).filter(Number.isInteger)
  };
}

async function hashPrompt(prompt) {
  const digest = await crypto.subtle.digest("SHA-256", new TextEncoder().encode(prompt));
  return `sha256:${[...new Uint8Array(digest)].map(byte => byte.toString(16).padStart(2, "0")).join("")}`;
}
```

The line immediately above `buildRunManifest` is the closing brace of `normalizeUrl`:

```js
function normalizeUrl(url) {
  try {
    const text = String(url || "").trim();
    if (!text) return "";
    return new URL(/^[a-z]+:\/\//i.test(text) ? text : `https://${text}`).href;
  } catch {
    return "";
  }
}
```

The manifest is built in `runFanout` (`background.js:58`) and stored under both `jayhawk.activeRun` and `jayhawk.runs.<runId>` (`background.js:59`).

---

## 2. Manifest object fields (enumerated)

The object returned by `buildRunManifest` has exactly these fields — no spread; every field is set literally.

| Field | Type | Source | Mandatory? |
|---|---|---|---|
| `run_id` | string | `runId` arg (caller passes `message.runId \|\| makeRunId()`; format `jh_<YYYYMMDDHHMMSS>_<6-char-rand>`) | mandatory |
| `prompt_hash` | string | `await hashPrompt(prompt)` — `"sha256:" + lowercase-hex SHA-256` of the trimmed prompt | mandatory |
| `prompt_title` | string | `prompt` arg, whitespace-collapsed and `.slice(0, 80)` | mandatory (may be empty string) |
| `targets` | string[] | `sites.map(site => site.name \|\| site.url)` — display names with URL fallback | mandatory (may be empty array) |
| `target_tab_urls` | string[] | per-`opened` item: live `chrome.tabs.get(tabId).url`, falling back to `item.url`, falling back to `""`. Same length and order as `opened`. | mandatory |
| `pasted_count` | number (integer) | `results.filter(r => r.ok).length` | mandatory |
| `created_at` | string (ISO 8601) | `new Date().toISOString()` | mandatory |
| `next_operator` | string (literal `"Claude Tab"`) | hard-coded constant | mandatory |
| `requested_collection_mode` | string (literal `"poll_live_tabs"`) | hard-coded constant | mandatory |
| `group_id` | number \| null | `groupId` arg — Chrome tab-group id from `chrome.tabs.group`, or `null` if no tabs opened or grouping failed | nullable, always present |
| `target_tab_ids` | number[] | `opened.map(item => item.tabId).filter(Number.isInteger)` — only successfully created tabs | mandatory (may be empty array) |

No spread is used inside `buildRunManifest`. (Spread is used elsewhere on the inbound FANOUT message and on per-site `pasteIntoTab` results — see §3.)

---

## 3. Per-site `opened` / `results` shape

### `openSite` — full body (verbatim)

```js
async function openSite(site, windowId, openerTabId, sourceIndex, offset) {
  const props = { url: site.url, active: false };
  if (Number.isInteger(windowId)) props.windowId = windowId;
  if (Number.isInteger(openerTabId)) props.openerTabId = openerTabId;
  if (Number.isInteger(sourceIndex)) props.index = sourceIndex + offset + 1;
  try {
    const tab = await api(chrome.tabs, "create", props);
    return { name: site.name || site.url, url: site.url, tabId: tab.id, ok: false, submitted: false };
  } catch {
    return { name: site.name || site.url, url: site.url, ok: false, submitted: false, reason: "tab-error" };
  }
}
```

`opened[i]` element fields:

| Field | Type | Present | Source |
|---|---|---|---|
| `name` | string | always | `site.name` or fallback `site.url` |
| `url` | string | always | `site.url` (already normalized) |
| `tabId` | number | only on success | `tab.id` from `chrome.tabs.create` |
| `ok` | boolean | always (initial `false`; flipped by paste result) | literal `false` |
| `submitted` | boolean | always (initial `false`; flipped by paste result) | literal `false` |
| `reason` | string | only on tab-create failure | literal `"tab-error"` |

### `pasteIntoTab` — full body (verbatim)

```js
async function pasteIntoTab(tabId, prompt, name) {
  try {
    await api(chrome.scripting, "executeScript", { target: { tabId }, files: ["content-injector.js"] });
    const [frame] = await api(chrome.scripting, "executeScript", {
      target: { tabId },
      func: promptText => window.__jayhawkPaste
        ? window.__jayhawkPaste(promptText)
        : { ok: false, submitted: false, reason: "injector-missing" },
      args: [prompt]
    });
    return { name, tabId, ...(frame?.result || { ok: false, submitted: false, reason: "no-result" }) };
  } catch {
    return { name, tabId, ok: false, submitted: false, reason: "tab-error" };
  }
}
```

`results[i]` element fields (final per-site result, what `runFanout` returns and stores):

| Field | Type | Present | Source |
|---|---|---|---|
| `name` | string | always | passed through from `opened[i].name` |
| `tabId` | number | always when paste was attempted; absent when `opened[i].tabId` was missing (in that case the original `opened` element is returned unchanged — see `runFanout` line 49: `if (!item.tabId) return item;`) | from `opened[i].tabId` |
| `ok` | boolean | always | spread from `frame.result` (provided by `window.__jayhawkPaste` in `content-injector.js`) |
| `submitted` | boolean | always | same source as `ok` |
| `reason` | string | only on failure paths | one of `"injector-missing"`, `"no-result"`, `"tab-error"`, `"load-timeout"`, or any `__jayhawkPaste` reason |
| (extras from `__jayhawkPaste`) | varies | — | spread via `...(frame?.result || ...)` |

The load-timeout path in `runFanout`:

```js
const result = ready
  ? await pasteIntoTab(item.tabId, prompt, item.name)
  : { name: item.name, tabId: item.tabId, ok: false, submitted: false, reason: "load-timeout" };
```

So `results[i]` always has `{ name, tabId?, ok, submitted, reason? }`, plus extras spread from `__jayhawkPaste`.

---

## 4. Storage keys (full list, verified)

All `jayhawk.*` (dot namespace). No `jayhawk_*` keys exist in the extension codebase.

| Key | File:Line | Storage area | Purpose |
|---|---|---|---|
| `jayhawk.lastGroup` | `background.js:1` (`LAST_GROUP_KEY`); used at lines 46, 127–128 | `chrome.storage.local` | `{ groupId, tabIds }` of the previous run, used by `closePreviousTabs` |
| `jayhawk.activeRun` | `background.js:2` (`ACTIVE_RUN_KEY`); used at line 59 | `chrome.storage.local` | The most recent run's manifest |
| `jayhawk.runs.<runId>` | `background.js:3` (`RUN_KEY_PREFIX` + runId); used at line 59 | `chrome.storage.local` | Per-run manifest archive, keyed by `runId` |
| `jayhawk.sites` | `popup.js:1`, `sidepanel.js:1`, `drawer.js:4` | `chrome.storage.sync` (via `getSync`/`setSync` helpers) | The user's site list |
| `jayhawk.lastPrompt` | `popup.js:2`, `sidepanel.js:1`, `drawer.js:5` | `chrome.storage.sync` | Last prompt text |
| `jayhawk.closePrev` | `popup.js:3`, `sidepanel.js:1`, `drawer.js:6` | `chrome.storage.sync` | "Close previous tabs" preference |
| `jayhawk.hideAfterGo` | `sidepanel.js:1` (`HIDE_KEY`) | `chrome.storage.sync` | "Hide side panel after Go" preference (sidepanel-only) |

---

## 5. Site object shape

### `sanitizeSites` (in `background.js`, verbatim)

```js
function sanitizeSites(sites) {
  return (Array.isArray(sites) ? sites : [])
    .map(site => ({ name: String(site.name || "").trim(), url: normalizeUrl(site.url) }))
    .filter(site => site.url);
}
```

Site object after sanitization: `{ name: string, url: string }` — `enabled` is stripped before the manifest layer.

### `cloneDefaults` (popup.js and sidepanel.js, verbatim)

```js
function cloneDefaults() {
  return DEFAULT_SITES.map(site => ({ ...site }));
}
```

`DEFAULT_SITES` (popup.js:4–8 and sidepanel.js:2–6, identical):

```js
const DEFAULT_SITES = [
  { name: "ChatGPT", url: "https://chatgpt.com/", enabled: true },
  { name: "DeepSeek", url: "https://chat.deepseek.com/", enabled: true },
  { name: "Gemini", url: "https://gemini.google.com/app", enabled: true }
];
```

Sidepanel additionally defines (line 7):

```js
const CLAUDE_SITE = { name: "Claude", url: "https://claude.ai/new", enabled: true };
```

### Site fields

| Field | Type | UI requires | Manifest layer |
|---|---|---|---|
| `name` | string | optional (UI); fallback to URL host | preserved (display name) |
| `url` | string | required; `normalizeUrl` auto-prepends `https://` if no scheme | normalized https URL |
| `enabled` | boolean | UI only — controls whether site is included in `selectedSites()` | stripped by `sanitizeSites`; never reaches manifest |

---

## 6. Message envelope (extension internal)

Every `chrome.runtime.sendMessage` and `chrome.runtime.onMessage.addListener` call across the JS files. (`background.js:209` uses `chrome.tabs.sendMessage` via the `notify` helper — included because it's how the background fans messages back to the source tab.)

### Senders — `chrome.runtime.sendMessage`

| File:Line | `type` | Other fields |
|---|---|---|
| `popup.js:64` | `"FANOUT"` | `detach: true`, `runId`, `prompt`, `sites`, `closePrev`, `currentWindowId`, `sourceTabId`, `sourceIndex` |
| `sidepanel.js:98` | `"FANOUT"` | `runId`, `prompt`, `sites`, `closePrev`, `currentWindowId`, `sourceTabId`, `sourceIndex` (no `detach`) |
| `drawer.js:154` | `"FANOUT"` | `runId`, `prompt`, `sites`, `currentWindowId: null`, `closePrev` (no `detach`, no `sourceTabId`/`sourceIndex`) |
| `drawer.js:187` | `"OPEN_SIDE_PANEL"` | (none — empty payload aside from `type`) |

### Senders — `chrome.tabs.sendMessage` (background → source tab, via `notify`)

The `notify(tabId, message)` helper at `background.js:207–210` calls `chrome.tabs.sendMessage(tabId, message)`. Call sites in `runFanout`:

| File:Line | `type` | Other fields |
|---|---|---|
| `background.js:21` | `"FANOUT_DONE"` | `runId: message.runId`, `results: []` (top-level error path) |
| `background.js:35` | `"FANOUT_DONE"` | `runId`, `results: []` (empty-prompt early-return) |
| `background.js:39` | `"FANOUT_PROGRESS"` | `runId`, `text: "Opening tabs..."` |
| `background.js:55` | `"FANOUT_PROGRESS"` | `runId`, `text: "Pasted into N of M..."` |
| `background.js:61` | `"FANOUT_DONE"` | `runId`, `results`, `manifest` (the full manifest from §1) |

### Listeners — `chrome.runtime.onMessage.addListener`

| File:Line | Reacts to `type` | Notes |
|---|---|---|
| `background.js:4` | `"OPEN_SIDE_PANEL"`, `"FANOUT"` | Reads: `type`, `detach`, `runId`, `prompt`, `sites`, `sourceTabId`, `sourceIndex`, `currentWindowId`, `closePrev`, `windowId` |
| `sidepanel.js:39` | `"FANOUT_PROGRESS"` | Reads: `type`, `text`. Updates `ui.status.textContent`. |
| `drawer.js:62` (handler at line 180) | `"FANOUT_PROGRESS"`, `"FANOUT_DONE"`, `"JAYHAWK_OPEN_DRAWER"` | Reads: `runId` (gates everything to current run), `type`, `text`. `JAYHAWK_OPEN_DRAWER` triggers `openPanelOrFallback()`. **Note:** `JAYHAWK_OPEN_DRAWER` is consumed but never sent by any inspected JS file — likely sent from outside (action-click handler or removed sender). |

### sendResponse paths from the top-level listener (`background.js:4–25`)

- `OPEN_SIDE_PANEL` → `{ ok: true }` on success, `{ ok: false, error }` on failure
- `FANOUT` with `detach: true` → `{ ok: true, accepted: true, runId }`
- `FANOUT` (sync) → resolves the `runFanout` return value `{ ok: true, results, manifest }`, or on error `{ ok: false, results: [], error }`

### Complete message-type catalogue

| `type` | Direction | Payload |
|---|---|---|
| `FANOUT` | UI (popup/sidepanel/drawer) → background | `{ type, runId, prompt, sites, closePrev, currentWindowId, sourceTabId?, sourceIndex?, detach? }` |
| `OPEN_SIDE_PANEL` | drawer (content) → background | `{ type, windowId? }` |
| `FANOUT_PROGRESS` | background → source tab | `{ type, runId, text }` |
| `FANOUT_DONE` | background → source tab | `{ type, runId, results, manifest? }` (manifest only on success path) |
| `JAYHAWK_OPEN_DRAWER` | (external — not sent by inspected JS) | Consumed by `drawer.js:184` to call `openPanelOrFallback()` |

---

## Sample runManifest (synthesized example)

For reference / test fixtures. NOT a real captured run; constructed from the verified schema.

```json
{
  "run_id": "jh_20260502081912_a3f1c0",
  "prompt_hash": "sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
  "prompt_title": "Compare three model takes on the portable memory model",
  "targets": ["ChatGPT", "DeepSeek", "Gemini"],
  "target_tab_urls": [
    "https://chatgpt.com/c/abc-123",
    "https://chat.deepseek.com/a/chat/s/xyz-456",
    "https://gemini.google.com/app/def-789"
  ],
  "pasted_count": 3,
  "created_at": "2026-05-02T08:19:12.443Z",
  "next_operator": "Claude Tab",
  "requested_collection_mode": "poll_live_tabs",
  "group_id": 47,
  "target_tab_ids": [1863835940, 1863835941, 1863835942]
}
```

---

*End of extension runManifest extraction. Verbatim function bodies and tables sourced from `C:\Users\ClawDaddy\Documents\New project 2\` on 2026-05-02 via read-only file inspection. No code modified. Drop-in replacement for `extension-snapshot/EXTENSION_RUNMANIFEST_EXTRACTION_PENDING.md`.*
