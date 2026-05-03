# DO NOT TOUCH (Contract-bearing)

## Extension (must not rename)
- runManifest field names/values once established
- storage keys:
  - `jayhawk.sites`
  - `jayhawk.lastPrompt`
  - `jayhawk.closePrev`
  - `jayhawk.hideAfterGo`
  - `jayhawk.lastGroup`
  - `jayhawk.activeRun`
  - `jayhawk.runs.*`
- message types:
  - `FANOUT`
  - `FANOUT_PROGRESS`
  - `FANOUT_DONE`
  - `OPEN_SIDE_PANEL`
  - `JAYHAWK_OPEN_DRAWER`
- content-injector per-host SELECTORS map

## Replit Sidecar (must not rename)
- localStorage keys:
  - `jayhawk_huddles`
  - `jayhawk_custom_models`

Note: extension uses dot-namespace keys; Replit uses underscore keys. Do not silently converge.
