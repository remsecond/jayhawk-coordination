# Platform Readiness Audit 01 (Jayhawk / OBE)

Repo baseline: `remsecond/jayhawk-coordination` @ `8195470`.
Environment: OpenClaw `2026.4.12`.

## 1. Executive finding

**You have strong “build” hands (git commits, branch pushes via SSH deploy key, headless Chromium control, Node/Python automation, model-native PDF/image analysis), but weak “account-bound” hands (Gmail/Google auth, GitHub PR creation/merge without the operator’s browser session, Playwright/Puppeteer tooling, OCR/PDF CLI stack, and MCP bridges).** The highest-leverage upgrades are: (1) keep SSH deploy-key branch pushes, (2) add a low-friction PR-creation/merge pathway (Tab UI when available, or `gh` + auth when operator is present), and (3) establish a minimal Gmail auth plan (either IMAP via app password or a one-time browser login with persistent profile).

## 2. Current hands (verified)

### Git/GitHub
- `git` present (`2.47.3`).
- **SSH is available** (OpenSSH `10.0p2`).
- A repo-scoped **SSH deploy key** for `remsecond/jayhawk-coordination` is installed and works for pushes (branch pushes verified).
- Can create branches/commits, push branches, and provide compare URLs.

### Browser control
- System `chromium` present (Chromium `147.0.7727.101`).
- OpenClaw browser control exists and is configured to allow private-network CDP and allowlisted hostnames (safe subset checked from `/data/.openclaw/openclaw.json`).

### Automation runtime
- Node `v22.22.2` + npm `10.9.7` present.
- Python `3.13.5` + pip `25.1.1` present.

### Document handling (model-native)
- OpenClaw has native tools for PDF/image analysis (model-mediated), which can replace some OCR/PDF parsing needs when “extract + reason” is sufficient.

## 3. Missing hands (verified)

### GitHub-native tooling
- `gh` CLI **not installed** (so no `gh pr create/merge`, no `gh auth login`).
- No discovered MCP config in `/data/.openclaw` (no MCP bridge found by filename search).

### Browser automation libraries
- Playwright CLI not present.
- Python libraries not present: `playwright`, `selenium`.
- (Puppeteer not audited as a package dependency; no repo-local Node automation stack exists yet.)

### OCR / PDF CLI stack
- No `tesseract`, `pdftotext`, or `qpdf` installed.
- No Python libs detected for OCR/PDF parsing: `pytesseract`, `pdfminer`, `PIL`.

### Gmail / Google Workspace access
- No evidence of configured Gmail API OAuth, refresh tokens, or IMAP client config in this environment (only Telegram pairing allowlists visible in `/data/.openclaw/credentials/`).

### Obsidian / vault access
- No direct vault connectivity is present here (no Obsidian filesystem surfaced, no vault API/tunnel configured).

## 4. Best upgrades (most useful new hands)

1) **Install `gh` CLI** (adds PR creation, PR status, issue workflows). Biggest reduction in operator relay for GitHub workflows when credentials are available.
2) **Standardize SSH deploy-key for branch pushes** (already working). Keep it repo-scoped.
3) **Add a “PR actions” pathway**:
   - When Tab/operator browser session is available: UI merge.
   - When not: `gh` can open PRs and post links, but merge remains operator-only if that rule is locked.
4) **Browser automation upgrade path**: install Playwright (Node or Python) only when you need DOM automation beyond OpenClaw’s built-in browser tool.
5) **Gmail access plan** (see Sections 5–6): pick one method and make it durable.

## 5. Gmail readiness (minimal setup plan)

### What can be done now (no operator auth)
- Prepare scripts, docs, and placeholders for Gmail workflows.
- Add allowlist hostnames (already includes `mail.google.com` and `accounts.google.com`).

### What requires credentials/operator involvement
- Any real Gmail access (IMAP credentials or OAuth) requires operator-provided secrets and/or a logged-in browser profile.

### Minimal viable plan (pick one)

**Option A: IMAP via app password (fastest, least machinery).**
- Operator creates a Gmail app password (requires 2FA) and enables IMAP.
- Configure an IMAP client (e.g. Himalaya-based skill) to read/send.
- Pros: durable, automation-friendly.
- Cons: app passwords are sensitive; must be stored/rotated carefully.

**Option B: Browser-login once, then reuse persistent profile (best when IMAP is undesired).**
- Operator logs into Gmail in the OpenClaw-controlled Chromium profile once (2FA interactive).
- Subsequent automation uses the persisted browser profile cookies.
- Pros: no IMAP/App Password.
- Cons: more brittle; headless/anti-bot risk; session expiry.

Recommendation: **Option A** if you want reliable programmatic email; **Option B** if you only need browser-driven actions.

## 6. Browser automation readiness (minimal setup plan)

### What works now
- OpenClaw browser tool + system Chromium are present.
- Safe domain allowlist exists; private-network CDP is enabled.

### What’s missing
- A general-purpose automation library (Playwright/Puppeteer) for scripted multi-step flows outside the OpenClaw browser tool.

### Minimal plan
- Start with OpenClaw browser tool for UI tasks.
- Only install Playwright when you hit a task that needs:
  - robust selectors and retries,
  - file downloads/uploads handling,
  - complex multi-tab orchestration,
  - full-page screenshots/PDF exports in bulk.

## 7. GitHub coordination readiness (minimal setup plan)

### What can be done right now without Roberto present
- Create branches and commits in `jayhawk-coordination`.
- Push branches via SSH deploy key.
- Provide compare URLs for PR creation.

### What can be prepared but not executed without Roberto
- PR creation/merge if GitHub UI is required and `gh` is unavailable/not authenticated.

### Minimal plan
- Keep using: branch push → compare URL → operator merges when available.
- Upgrade later: install `gh` to create PRs (merge policy remains operator-controlled).

## 8. MCP / tool bridge readiness

- No MCP bridge/config was discovered by filename search in `/data/.openclaw`.
- If MCP is desired, the least risky bridge is GitHub-only (PRs/issues) and should not be treated as a general orchestrator.

## 9. Security cautions

- Deploy keys have no passphrase: if the host is compromised, repo write access is compromised. Keep keys repo-scoped and removable.
- Treat any token pasted into chat as burned. Revoke/rotate immediately.
- Avoid storing Google credentials in plaintext. Prefer OpenClaw credential storage and least privilege.

## 10. Recommended next setup sequence

1) Install `gh` CLI (no auth yet), so PR creation becomes possible once operator auth exists.
2) Decide Gmail approach (IMAP/app-password vs persistent browser profile) and do the one-time operator auth step.
3) Defer Playwright/Puppeteer until a concrete task demands it.
4) Keep GitHub coordination on: push branch → compare URL → operator merges (or Tab merges when available).
