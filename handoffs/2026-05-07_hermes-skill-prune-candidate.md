---
title: Hermes Skill Prune — Candidate Disable List
status: candidate (pre-config)
type: operations-recommendation
owner: roberto (Tab)
authors:
  - dispatch (claude)
created: 2026-05-07
last_updated: 2026-05-07
related:
  - handoffs/2026-05-07_hostinger-cleanup-next-steps.md (item 2)
  - handoffs/2026-05-07_hermes-openclaw-integration-findings.md
note: "Drafted from the Hermes Agent v0.9.0 startup banner, before config.yaml has been printed. Diff this list against the actual config.yaml when it arrives and reconcile."
---

# Hermes Skill Prune — Candidate Disable List

## Hermes's role on this box

Hermes is **Claw's coordinating observer**. Her load-bearing jobs are:
1. Read OpenClaw's published state on the shared surface (sessions, tasks, workspace).
2. Write to her own outbound dir (`/coordination/from-hermes/`) so Claw can see her signals.
3. Send/receive Telegram messages on her own bot identity.
4. Run small admin/operations work (cron-style watchdogs, file edits, terminal commands).
5. Optionally: notes, web research, GitHub interactions in support of (1)–(4).

Anything that doesn't serve those five jobs is prunable. The prune is a `config.yaml` edit; it reduces prompt-injection surface, credential-exposure surface, and Hermes's startup time.

## Source of truth (so far)

This list is drafted from the **Hermes Agent v0.9.0 startup banner** as captured 2026-05-07 (28 tools, 75 skills). The banner truncates several cluster listings (`creative:`, `mlops:`, `productivity:`, `research:`, `software-development:`, `github:`) — the truncated entries need to be filled in once `config.yaml` prints. Treat anything below marked `(truncated — fill from config)` as TBD.

## Toolsets (28 total visible)

### Hard keep — load-bearing for the role

| Toolset | Reason |
|---|---|
| `file` (`patch`, `read_file`, `search_files`, `write_file`) | core — read shared surface, write her own outbound dir |
| `terminal` | core — admin commands, recon, the kind of work she did 2026-05-07 |
| `code_execution` (`execute_code`) | needed for jq/sed/python parsing of session JSONLs |
| `cronjob` | watchdog implementation per cleanup doc item 4 |
| `delegation` (`delegate_task`) | spawn subagents for parallel reads/research |
| `memory` | session continuity across resumes |
| `messaging` | Telegram bot (per v0.1 §11.1) |
| `session_search` | resume + audit prior recon |
| `todo` | task tracking — already used in the recon discipline |
| `browser` | web fetches when running research handoffs |
| `clarify` | interaction quality |
| `mcp` | future MCP integrations (don't pre-prune optionality here) |

### Probable prune — depends on Tab's actual use

| Toolset | Reason to consider pruning | Decision (2026-05-07) |
|---|---|---|
| `homeassistant` (`ha_call_service`, `ha_get_state`, …) | only useful if Tab actually runs Home Assistant | **Keep**. Tab: "no but may." Optionality is cheap; the cluster doesn't hold credentials by default. |
| `image_gen` (`image_generate`) | not relevant to coordination work; reduces image-output prompt-injection surface | **Prune**. |
| `tts` | only matters if Hermes outputs audio | **Prune**. Tab: "no idea" — default to remove. Trivially restorable later if Tab decides he wants voice. |
| `vision` | image ingestion | **Keep** — useful for screenshot debugging when Tab pastes images into Hermes. |

### Hard prune — definitely not load-bearing

| Toolset | Reason |
|---|---|
| (none called out as toolset; most prunable items are at skill level — see below) | |

## Skills (75 total, organized by cluster)

### Hard prune

| Cluster | Skills | Reason |
|---|---|---|
| `red-teaming` | `godmode` | named in cleanup doc item 2; reduces injection/exfil surface; Hermes has zero red-team mandate |
| `gaming` | `minecraft-modpack-server`, `pokemon-player` | not coordination-relevant; named in cleanup doc |
| `media` | `gif-search`, `heartmula`, `songsee`, `youtube-content` | not coordination-relevant; consumer-media surface |
| `mlops` (subset) | `audiocraft-audio-generation`, `axolotl` | model-training and audio gen — out of scope; named in cleanup doc |
| `smart-home` | `openhue` | not coordination-relevant; redundant with `homeassistant` toolset if HA stays |
| `social-media` | `xitter` | reduces public-write surface |
| `leisure` | `find-nearby` | not coordination-relevant |
| `creative` (subset) | `ascii-art`, `ascii-video` | novelty; not load-bearing |

### Probable prune — confirm before disabling

| Cluster | Skills | Why questionable |
|---|---|---|
| `creative` | `architecture-diagram`, others *(truncated — fill from config)* | architecture-diagram could be useful for design work; others likely prunable |
| `mlops` | `clip`, `dsp...` *(truncated — fill from config)* | depends on whether Hermes ever does ML side-quests |

### Hard keep — load-bearing or aligned with role

| Cluster | Skills | Why keep |
|---|---|---|
| `autonomous-ai-agents` | `claude-code`, `codex`, `hermes-agent`, `opencode` | needed for delegation + coordination semantics with Claw |
| `data-science` | `jupyter-live-kernel` | useful for session-transcript analysis |
| `devops` | `webhook-subscriptions` | future cleanup tooling, contract validation |
| `email` | `himalaya` | possible operator channel; cheap to keep |
| `general` | `dogfood` | self-test discipline |
| `github` | `codebase-inspection`, `github-auth`, `github-code-r...` *(truncated)* | repo work for the jayhawk-coordination pipeline |
| `mcp` | `mcporter`, `native-mcp` | MCP optionality |
| `note-taking` | `obsidian` | Tab uses Obsidian — directly load-bearing for the case-file discipline |
| `productivity` | `google-workspace`, `linear`, `nano-pdf`, `notion`, `ocr...` *(truncated)* | most are likely keepers; review per item once config prints |
| `research` | `arxiv`, `blogwatcher`, `llm-wiki`, `polymarket`, `resea...` *(truncated)* | research handoffs per finding 5 of integration findings doc |
| `software-development` | `plan`, `requesting-code-review`, `subagent-driven-d...` *(truncated)* | core to her contribution to v0/v0.1 work |

## What's still uncertain (resolves when config.yaml prints)

1. **Truncated cluster listings**: `creative`, `mlops`, `productivity`, `research`, `software-development`, `github` all show `…` in the banner. Fill from config. Tab on this 2026-05-07: "not sure" — proposed approach is item-by-item review when each cluster's full skill list is visible, defaulting to keep unless I can articulate a specific reason to prune.
2. ~~`homeassistant` toolset usage~~ **Resolved 2026-05-07**: keep (Tab: "no but may" → preserve optionality).
3. ~~`tts` / `vision`~~ **Resolved 2026-05-07**: tts → prune; vision → keep.
4. **Exact YAML structure**: skills may be listed under `enabled:` / `disabled:` arrays, by cluster, or via per-skill blocks. The diff format depends on Hermes's actual config schema, which we haven't seen yet.

## Update 2026-05-07: skills are filesystem-based, not YAML-managed

The Hermes operator session printed `/opt/data/config.yaml` and confirmed it is **three lines, 56 bytes**:

```yaml
model:
  provider: anthropic
  default: claude-opus-4-7
```

That's the entire file. Skills are **not declared in YAML at all** — Hermes discovers them by scanning `/opt/data/skills` and plugin dirs. API credentials aren't in this file either; they come from env vars (per Phase 2's `.env` findings).

This changes the prune approach materially:

- **Wrong artifact**: a YAML diff. There's nothing to diff.
- **Right artifact**: a top-level directory listing of `/opt/data/skills` and any plugin dirs, then per-directory keep-or-cut decisions. The "patch" becomes "remove these directories" (or rename them out of the discovery path) rather than "comment out these YAML keys."

### Revised process

1. **Recon prompt to Hermes** (single-line, no recursion, no cat):
   ```
   ls -la /opt/data/skills /opt/data/plugins 2>/dev/null;
   echo ---;
   find /opt/data -maxdepth 3 -type d -name skills 2>/dev/null
   ```
2. **Categorize each top-level skill directory** against the buckets in this doc (hard keep / probable prune / hard prune).
3. **Resolve any directories** I haven't anticipated above by asking Tab item-by-item.
4. **Propose a removal plan**: list of dirs to `rm -rf` (or rename to e.g. `skills.disabled/<name>` if reversibility matters more than disk space).
5. **Tab approves**; the operator applies via Hermes's `file` toolset, restarts Hermes, verifies via fresh `/help` that the skill count dropped to the expected number.
6. **Land the applied diff** in `handoffs/2026-05-07_hermes-skill-prune-applied.md`, including before/after skill counts and the actual command(s) run.

### What this does NOT change

The keep/probable-prune/hard-prune buckets above are still valid — they categorize by skill purpose, not by config format. Use them as the categorization framework when the directory listing arrives.

## What this does NOT do

- Does not apply any change. This file is a **candidate**, not a directive.
- Does not assume the truncated banner items above. Anything `(truncated — fill from config)` is explicitly TBD.
- Does not touch tools/skills used by OpenClaw — Claw's config is separate and is not in scope here.
- Does not modify Hermes's `.env` or credential files. Skill prune only.

---

*End of candidate prune list. Pair with `config.yaml` once it prints; produce the applied-diff handoff after Tab signs off on the patch.*
