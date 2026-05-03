# Replit Sidecar App — App.tsx schema extraction (verbatim)

**Source:** `artifacts/sidecar/src/App.tsx` (Replit Sidecar app, deployed at `jayhawk-sidecar-control.replit.app`)
**Extracted:** 2026-05-02 by Replit AI assistant via read-only file inspection; relayed to Dispatch by Roberto
**Purpose:** drop-in for `replit-sidecar-snapshot/` in the `jayhawk-coordination` repo (parallel to `extension-snapshot/`)
**Read-only.** No code changes; this is a snapshot for the contract-first integration plan.

---

## 1. Type definitions (verbatim)

### Huddle (lines 15–27) — the durable record

```ts
type Huddle = {
  id: string;
  status: HuddleStatus;
  compiledPrompt: string;
  outcomeSnapshot: string;
  instructionsSnapshot: string;
  modesSnapshot: string[];
  presetIdsSnapshot: string[];
  targets: Target[];
  host: HostChoice;
  hostDirective: string;
  createdAt: number;        // epoch milliseconds via Date.now()
};
```

### Status enums

```ts
// line 12
type HuddleStatus = "active" | "complete" | "abandoned";

// line 9 — UI-only; not stored on the Huddle, drives transient dispatch animation
type DispatchStatus = "idle" | "preparing" | "opening" | "sent" | "failed";
```

### Other module-scope types

```ts
// line 7
type Model = { id: string; label: string; url?: string; custom?: boolean };

// line 8
type Preset = { id: string; label: string; prompt: string };

// line 11 — per-target lifecycle (distinct from HuddleStatus)
type TargetStatus = "pending" | "opened" | "failed" | "complete";

// line 13 — who synthesizes the host directive
type HostChoice = "GPT" | "Claude";

// line 14 — per-target object
type Target = { modelId: string; label: string; status: TargetStatus; response: string };

// lines 50–51 — UI theming, not contract-relevant
type ThemeSetting  = "system" | "dark" | "light";
type ResolvedTheme = "dark" | "light";
```

---

## 2. Storage keys (verbatim, lines 45–47)

All `localStorage` (single browser; no cross-device sync).

```ts
const HUDDLES_KEY       = "jayhawk_huddles";       // Huddle[] (capped by MAX_HUDDLES)
const CUSTOM_MODELS_KEY = "jayhawk_custom_models"; // user-defined Model[] additions
const THEME_KEY         = "jayhawk_theme";         // ThemeSetting (UI only)
```

---

## 3. Storage I/O functions (verbatim)

### `loadHuddles` (lines 163–171)

```ts
function loadHuddles(): Huddle[] {
  try {
    const raw = localStorage.getItem(HUDDLES_KEY);
    if (!raw) return [];
    const parsed = JSON.parse(raw);
    if (!Array.isArray(parsed)) return [];
    return parsed.filter((h) => h && typeof h.id === "string" && Array.isArray(h.targets));
  } catch { return []; }
}
```

### `saveHuddles` (lines 173–175)

```ts
function saveHuddles(huddles: Huddle[]) {
  try { localStorage.setItem(HUDDLES_KEY, JSON.stringify(huddles.slice(0, MAX_HUDDLES))); } catch {}
}
```

`MAX_HUDDLES` value: not in the extracted snippets. Implies a stored history cap; ask Replit AI to dump the constant when convenient.

---

## 4. Compiled prompt (verbatim, lines 403–447)

`compiledPrompt` is constructed inline as an IIFE in render scope. The output is stored on the Huddle as `compiledPrompt: string`.

```ts
const compiledPrompt = (() => {
    if (!hasOutcome && !hasInstructions) return "";

    const activeModes = presets.filter((p) => activePresets.includes(p.id)).map((p) => p.label);

    const lines: string[] = [];
    lines.push("# Jayhawk dispatch");
    lines.push(
      orderedTargetLabels.length > 1
        ? `You are one of ${orderedTargetLabels.length} models receiving this prompt in parallel.`
        : `You are receiving this prompt as part of a multi-model dispatch.`
    );
    lines.push(`Targets: ${orderedTargetLabels.length ? orderedTargetLabels.join(", ") : "(none selected)"}`);
    lines.push("");

    if (activeModes.length > 0) {
      lines.push(`Modes: ${activeModes.join(", ")}`);
      lines.push("");
    }

    if (hasOutcome) {
      lines.push("## Outcome");
      lines.push(taskPrompt.trim());
      lines.push("");
    }

    if (hasInstructions) {
      lines.push("## Instructions");
      lines.push(discussionPrompt.trim());
      lines.push("");
    }

    lines.push("---");
    lines.push("Respond directly. Do not coordinate with other models. Do not assume shared context.");

    // Dispatch metadata (not model-facing prose, kept terse for inspection)
    const meta: string[] = [];
    if (closeTabs) meta.push("close-previous-tabs=true");
    if (meta.length > 0) {
      lines.push("");
      lines.push(`<!-- dispatch: ${meta.join(" ")} -->`);
    }

    return lines.join("\n").trimEnd();
  })();
```

### Compiled prompt structure (the output shape)

```
# Jayhawk dispatch
You are one of N models receiving this prompt in parallel.    (or "as part of a multi-model dispatch" if N=1)
Targets: ChatGPT, DeepSeek, Gemini

Modes: <preset labels comma-separated>                         (only if any presets active)

## Outcome                                                      (only if outcome present)
<outcome text trimmed>

## Instructions                                                 (only if instructions present)
<instructions text trimmed>

---
Respond directly. Do not coordinate with other models. Do not assume shared context.

<!-- dispatch: close-previous-tabs=true -->                     (only if close-tabs flag on; one or more meta tokens)
```

---

## 5. Huddle lifecycle functions (verbatim)

### `handleGo` — creates a new Huddle (lines 568–617)

```ts
function handleGo() {
    if (!canGo) return;

    const huddleId = uid();
    // Use chip-row order so huddle targets, Preview, and chips all agree.
    // Initialize targets directly as "opened" so the row label reads "Awaiting
    // paste" from the first frame — no PENDING flicker.
    const targets: Target[] = orderedSelectedModels.map((m) => ({
      modelId:  m.id,
      label:    m.label,
      status:   "opened",
      response: "",
    }));
    const modesSnapshot = presets.filter((p) => activePresets.includes(p.id)).map((p) => p.label);

    // Downgrade any prior active huddle BEFORE creating the new one
    reconcilePriorActive(huddleId);

    setHuddle({
      id:                   huddleId,
      status:               "active",
      compiledPrompt:       compiledPrompt,
      outcomeSnapshot:      taskPrompt.trim(),
      instructionsSnapshot: discussionPrompt.trim(),
      modesSnapshot,
      presetIdsSnapshot:    [...activePresets],
      targets,
      host:                 "GPT",
      hostDirective:        "",
      createdAt:            Date.now(),
    });
    setHuddlePromptOpen(false);
    setDirectiveCopied(false);
    setPromptCopied(false);

    setDispatchStatus("preparing");

    // Auto-scroll to the Active Huddle so the user sees the next-step strip
    setTimeout(() => {
      huddleRef.current?.scrollIntoView({ behavior: "smooth", block: "start" });
    }, 60);

    setTimeout(() => {
      setDispatchStatus("opening");
      setTimeout(() => {
        setDispatchStatus("sent");
        setTimeout(() => setDispatchStatus("idle"), 3000);
      }, 900);
    }, 450);
  }
```

### `setHostDirective` (lines 640–643)

```ts
function setHostDirective(text: string) {
    setHuddle((prev) => prev ? { ...prev, hostDirective: text } : prev);
    setDirectiveCopied(false);
  }
```

### `copyDirective` — completes the Huddle (lines 649–659)

```ts
async function copyDirective() {
    if (!huddle?.hostDirective.trim()) return;
    try {
      await navigator.clipboard.writeText(huddle.hostDirective);
      setDirectiveCopied(true);
      setHuddle((prev) => prev ? { ...prev, status: "complete" } : prev);
      setTimeout(() => setDirectiveCopied(false), 1800);
    } catch {
      setDirectiveCopied(false);
    }
  }
```

---

## 6. Lifecycle behavior (derived from code)

### Huddle status transitions

| From | To | Trigger |
|---|---|---|
| (none) | `active` | `handleGo()` creates new Huddle with `status: "active"` |
| `active` | `complete` | `copyDirective()` succeeds (clipboard write OK) |
| `active` | `abandoned` | `reconcilePriorActive(newHuddleId)` called by `handleGo` for the previous active Huddle |

Invariant: at most ONE Huddle has `status: "active"` at a time (enforced by `reconcilePriorActive` running BEFORE the new active Huddle is created).

### Per-target status transitions

| From | To | Trigger |
|---|---|---|
| (n/a) | `opened` | `handleGo()` initializes all targets as `opened` (skipping `pending` to avoid UI flicker) |
| `opened` | `complete` | (presumed) when user pastes a `response` for that target — extraction does not include this code path |
| `opened` | `failed` | (presumed) error path — extraction does not include this code path |

Note: `pending` is in the enum but never actually used in `handleGo` per the inline comment. May be set by a code path not extracted.

### DispatchStatus animation (UI-only, not stored)

`handleGo` drives a transient animation:

```
preparing  → (450ms) → opening → (900ms) → sent → (3000ms) → idle
```

`DispatchStatus` is NEVER persisted on the Huddle. It's a render-state hint only.

### Completion semantics

The Huddle moves `active → complete` ONLY when the host directive is successfully copied to the clipboard. There is no separate "save directive" step — the copy IS the completion signal. If the clipboard write fails, the Huddle remains `active`.

---

## 7. Sample Huddle (synthesized example)

For reference / test fixtures. NOT a real captured Huddle; constructed from the verified schema.

```json
{
  "id": "h_8c3f1a2b",
  "status": "complete",
  "compiledPrompt": "# Jayhawk dispatch\nYou are one of 3 models receiving this prompt in parallel.\nTargets: ChatGPT, DeepSeek, Gemini\n\nModes: Critical Review\n\n## Outcome\nIdentify the strongest objection to the portable memory architecture.\n\n## Instructions\nFocus on dissent. Avoid consensus moves.\n\n---\nRespond directly. Do not coordinate with other models. Do not assume shared context.",
  "outcomeSnapshot": "Identify the strongest objection to the portable memory architecture.",
  "instructionsSnapshot": "Focus on dissent. Avoid consensus moves.",
  "modesSnapshot": ["Critical Review"],
  "presetIdsSnapshot": ["preset_critical_review"],
  "targets": [
    { "modelId": "m_chatgpt", "label": "ChatGPT", "status": "complete", "response": "<verbatim ChatGPT response>" },
    { "modelId": "m_deepseek", "label": "DeepSeek", "status": "complete", "response": "<verbatim DeepSeek response>" },
    { "modelId": "m_gemini", "label": "Gemini", "status": "complete", "response": "<verbatim Gemini response>" }
  ],
  "host": "Claude",
  "hostDirective": "Synthesized directive text — the user-edited consensus or chosen position.",
  "createdAt": 1746162000000
}
```

---

## 8. Notes for the shared contract

- `createdAt` is `number` (epoch ms), NOT ISO string. Extension uses ISO string for `created_at`. Contract must reconcile.
- `host` is the SYNTHESIZER (who writes the directive AFTER responses arrive). Extension's `next_operator` is the COLLECTOR (who gathers responses BEFORE synthesis). These are distinct lifecycle roles even though both name an agent.
- `compiledPrompt` is the dispatch-time snapshot of the prompt — including the model-facing "Respond directly. Do not coordinate..." footer. Extension transmits raw prompt text to model surfaces; Replit constructs and stores the wrapped version.
- `Target.response: string` is the verbatim user-pasted response. Extension never sees this — collection happens outside the extension boundary.
- `presetIdsSnapshot` AND `modesSnapshot` are both stored — IDs for re-rendering UI, labels for display. The dual storage is intentional (preset definitions can change; snapshot freezes both views).

---

*End of Replit App.tsx extraction. Verbatim from `artifacts/sidecar/src/App.tsx` lines 7–14, 15–27, 45–47, 50–51, 163–175, 403–447, 568–617, 640–643, 649–659. Drop-in for `replit-sidecar-snapshot/` in `jayhawk-coordination`.*
