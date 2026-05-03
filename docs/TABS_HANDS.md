# Tab’s hands (optional)

Tab is an **optional** operator-adjacent surface that can do UI actions when it’s available.
The system must still work without Tab.

## What Tab can do (when present)

- Open GitHub PR pages and perform UI actions (create PR, review, merge, delete branch), under operator supervision.
- This is useful when API/CLI auth for PR creation is unavailable from other actors.

## What to do when Tab is NOT present

- Use patch/branch flow only:
  - actors produce patches or push branches (if allowed)
  - operator opens/merges PRs later when they have GitHub access

## Rule

- GitHub UI actions are executed in Tab **only when available**.
- Tab is never a requirement for the coordination pattern to function.
