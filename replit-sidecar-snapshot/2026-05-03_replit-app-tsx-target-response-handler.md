---
title: Replit Sidecar App — MAX_HUDDLES + per-target response handler (verbatim)
source: artifacts/sidecar/src/App.tsx
extracted: 2026-05-03
notes: "Read-only snapshot, relayed by Big Daddy. Complements 2026-05-02_replit-app-tsx-extraction.md."
---

# Replit Sidecar App — MAX_HUDDLES + per-target response handler (verbatim)

## MAX_HUDDLES constant

```ts
// §10 Q2 — MAX_HUDDLES constant (artifacts/sidecar/src/App.tsx, line 48)
const MAX_HUDDLES = 12;
```

## Per-target paste handler: setTargetResponse

```ts
// §10 Q3 — per-target paste handler: setTargetResponse (lines 619–638)
function setTargetResponse(modelId: string, text: string) {
  setHuddle((prev) => {
    if (!prev) return prev;
    return {
      ...prev,
      targets: prev.targets.map((t) => {
        if (t.modelId !== modelId) return t;
        const trimmed = text.trim();
        // Require more than a single character before flipping to "complete"
        // — stray keystrokes shouldn't count, but legitimately short replies
        // like "Yes." or "N/A" should.
        const meaningful = trimmed.length >= 2;
        const nextStatus: TargetStatus = meaningful
          ? "complete"
          : t.status === "complete" ? "opened" : t.status;
        return { ...t, response: text, status: nextStatus };
      }),
    };
  });
}
```

## Wiring (textarea onChange)

```ts
// Wiring at the per-target textarea (line ~1602–1606):
// <textarea
//   data-testid={target-response-${t.modelId}}
//   ...
//   onChange={(e) => setTargetResponse(t.modelId, e.target.value)}
// />
```
