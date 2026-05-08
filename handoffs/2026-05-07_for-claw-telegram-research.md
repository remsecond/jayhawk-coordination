# For Claw — Telegram research and decision context

> Verbatim capture from `For Claw.txt` (Roberto's Downloads, 2026-05-07 15:43:32, 13,875 bytes). Handoff produced by the operator Claude session (the one driving Hermes prompts) before writing any Telegram code. Lands here per repo rule "if it matters, it lands here as a file + commit." Pair with `contracts/SHARED_SURFACE_HOSTINGER_V0_1.md` §11.1, which has been updated to incorporate the operational rules below.

---

Quick housekeeping before I dive into Telegram research.

About the "decisions in order" block at the top of your message. That's quoting another Claude session — references things I have no context on tonight: "Tab Claude's three offers," "Contract §11," "§7 verification," "PR: draft branch holds," "Finding 2 disagreement," "allowlist vs double-lock." I don't want to act on those as if they were instructions to me, because I genuinely don't know what they refer to and I don't want to make up the missing context. If those decisions are meant to apply here, send me whatever document/branch they're scoped to (the draft contract, the §11 you're referring to, the §7 spec) and I'll align. If they're notes from a parallel thread you're keeping in another window and you just shared them as context, I'll read them as context only — no action. Tell me which.

Direct restatement of where we agreed in this thread:

Messaging layer covers both human↔agent and agent↔agent. Two layers, one mental model.

Hermes needs a write-back path. Second bind-mount required (RW into Hermes, RO into OpenClaw, scoped to a single new directory).

Credentials separation is the right shape; you asked whether we have to do it now. Answer: no, not tonight. Here's the dependency chain so you know when it does have to happen. The current `telegram-pairing.json` is protected today by mode 0700 + RO mount — that combination is a fail-open default, but it is currently holding because Hermes runs as a different uid and physically can't read it. That stays true as long as (a) Hermes's uid doesn't change, (b) we don't flip the mount to RW, and (c) we don't add new credential files that aren't 0700. The moment any of those three change, the layered defaults stop holding. Practically: if we add a Telegram bot for Hermes, that bot's token is a new credential. Don't put it in the shared `credentials/` dir. Put it in `/docker/hermes-agent-bnz0/data/.private/hermes-telegram.token` (Hermes-only, never shared) and document that path in §11. The OpenClaw token stays where it is in `credentials/telegram-pairing.json`. So the answer is: don't move what's already there, but don't add to it either. The migration to a private credentials dir can wait until you have a third agent or off-box deployment.

Now to your real ask: Telegram research before we write code.

## Telegram bot research, mapped to our two-agent architecture

I'll cover the well-known patterns and the well-known failure modes, then point at the decisions you need to make for §11.

**Long-polling vs webhook.** Telegram offers two ways to receive updates. Webhook means Telegram pushes messages to a public HTTPS URL you operate. Long-polling means your bot calls `getUpdates` and Telegram holds the connection open until something arrives. On Hostinger, webhook is doable (you have a public IP and can terminate TLS via a reverse proxy or use Hostinger's hosted domain), but it adds a moving part: a publicly reachable endpoint with a valid cert, behind which your bot lives. Long-polling needs zero inbound network — the bot makes outbound HTTPS calls to `api.telegram.org` and that's it. For our setup, where both agents already make outbound HTTPS for the Anthropic API, long-polling is the obviously simpler choice. Webhook becomes attractive only when you have many bots or very high message volume. We have neither. **Recommend long-polling.**

**The update_id offset discipline.** This is the most common thing that goes wrong in DIY Telegram bots. `getUpdates` returns updates with monotonically increasing `update_id`s. You're supposed to call `getUpdates` next time with `offset = last_update_id + 1` so Telegram knows you've consumed them. People forget, or they crash mid-batch, and they end up reprocessing the same updates forever. The fix is to **persist the last consumed update_id to disk before you act on the update, not after.** OpenClaw's `telegram/update-offset-default.json` (348 bytes, mtime 2026-05-01) is exactly this file — they're already doing it right. Hermes should follow the same pattern in her own scope: a single small JSON file `/opt/data/telegram-offset.json` storing the last update_id she's consumed. This makes the bot crash-safe and replay-safe.

**Rate limits.** Documented limits: ~30 messages/sec across different chats, 1 message/sec to the same chat, 20 messages/minute to the same group chat. Bulk-broadcast scenarios will trip these. For human↔agent traffic, none of these matter — you don't message yourself faster than 1/sec. For agent↔agent traffic routed through Telegram, you'd hit them quickly during any state-sync burst. **This is one of the strongest arguments for keeping agent↔agent off Telegram and on the filesystem bus.** If you do route some agent traffic through it (e.g., "Hermes sends OpenClaw a structured request and waits for reply"), the pattern is: respect `retry_after` in the 429 response (Telegram tells you exactly how long to wait), implement exponential backoff for everything else, and never tight-loop.

**Idempotency.** Telegram retries deliveries on its end if it doesn't get an HTTP 200 back from your webhook (irrelevant for us since we're long-polling) or if the `getUpdates` connection drops mid-flight (relevant). The clean idempotency key is `(chat_id, message_id)`. Two messages with the same `(chat_id, message_id)` are the same logical event. If you store consumed events keyed on that pair, retries are harmless. Same applies to outbound: if your bot posts a reply and the network blips, you re-post — but tag your outbound with a logical key so you can detect "I already posted this" and skip. This is more important for agent traffic than for human conversations.

**Single router-bot vs per-agent bots.** Two patterns, both documented.

- **Per-agent bots**: each agent has its own bot token. Hermes is `@HermesAgent_bot`, OpenClaw is `@OpenClawAgent_bot`. You add both to your phone. Pro: clean identity, you know who's pinging you from the chat header, easy to mute one without muting the other, easy to revoke one token without affecting the other. Con: two tokens to manage, two `getUpdates` loops, two offset files. Each agent only ever sees its own messages.
- **Single router bot**: one bot token, one chat, both agents share it. The bot's "personality" is the union; you tell who's talking via prefix or formatting. Pro: one token, one chat, simpler. Con: no identity separation in chat headers; if one agent is compromised, all bot traffic is compromised; revocation is all-or-nothing.

For two agents on one box that you want to keep cleanly separated, **per-agent bots is the better hygiene**. The naming you already have (`telegram-pairing.json` for OpenClaw, future `hermes-telegram.token` for Hermes) supports this. The slight extra ops cost is worth the identity isolation.

**MarkdownV2 escaping.** This is a documented landmine. Telegram supports two parse modes: `Markdown` (legacy, lenient, deprecated) and `MarkdownV2` (current, strict). MarkdownV2 requires escaping these characters: `_ * [ ] ( ) ~ \ > # + - = | { } . !` — yes, the period and the exclamation mark too. If you forget, your `sendMessage` call returns a 400 with "can't parse entities." Most agent chat output contains periods and parens by accident. **The well-known fix is: don't use MarkdownV2 unless you actually need formatting; default to plain text. If you do need formatting, use `HTML` parse mode**, which only requires escaping `< > &` and is much friendlier. For agent-generated text that may include code blocks, use a known-safe escaper from your library (most have one) and never hand-build the string.

**File size limits.** Bots can send files up to 50 MB and receive up to 20 MB through the standard API. There's a separate local Bot API server mode that lifts these but requires running your own server. For our scope (text messages, occasional small attachments), the defaults are fine. If you ever want OpenClaw to send a multi-MB report through Telegram, plan for chunking or a presigned URL pattern — don't push against the limit.

**Group chat gotchas.** If the Telegram chat where the agents post is a group (not a private 1:1 with the bot), there's a documented gotcha called **privacy mode**. In privacy mode (the default for new bots), the bot only receives messages addressed to it — `/command` or `@BotName` mention. Plain group chatter is invisible to the bot. To see all messages in a group, you have to disable privacy mode via `@BotFather`. People hit this and assume the bot is broken. **Decision point**: are agent posts going to a private 1:1 chat with each bot, or a single group chat with both bots and you? Private 1:1 is simpler. Group chat is nice for "everyone sees everything" but requires the privacy-mode flip.

**Library landscape.** Three serious choices for the implementation:

- `python-telegram-bot` (Python, async, well-maintained, large community, the de facto reference). If Hermes or Claw are running in Python or have a Python sidecar, this is the safe pick.
- `Telegraf` (Node.js, used by a lot of production bots, idiomatic Node middleware pattern). If your stack is Node — and OpenClaw's `node_modules/openclaw` we saw tonight suggests it might be — Telegraf is the natural choice.
- `grammY` (Node.js, newer, cleaner API than Telegraf, growing community). Worth considering if you're starting fresh.

**Avoid raw axios calls to `api.telegram.org`.** Yes, the API is simple enough to use directly, and people do it for one-off scripts, but the libraries above handle offset persistence, rate-limit backoff, retries, and parse-mode escaping for you. Reinventing those is exactly the kind of thing that bites at 3 AM.

## The "well-known issues" that will bite us

1. **The bot stops receiving updates** because someone called `setWebhook` once during testing and never called `deleteWebhook` to switch back to long-polling. Telegram silently routes everything to the dead webhook. **Fix: explicit `deleteWebhook` at startup if you're committing to long-polling.**
2. **Two instances of the same bot** (e.g., two Hermes containers, or one running plus one in a debugger) racing on `getUpdates` with the same token. They alternate consuming each other's updates and you lose half. **Fix: never run two processes with the same bot token. If you need to test, use a separate test bot.**
3. **The agent crashes mid-reply**, then on restart re-reads the same `update_id` and replies again. User sees duplicate messages. **Fix: persist offset before acting; idempotency-tag outbound messages.**

---

## File-search context (separate Claude pass)

> The above content was located via a file-search Claude pass: scanned Downloads recursively, Documents top level, and Desktop top level. Expanded to a 48-hour window because the May 7-only pass was sparse. The exact `C:\Users\ClawDaddy\Documents\moyer_case` path does not exist on this machine, and no similarly named top-level Documents folder showed up. Desktop had zero files modified on May 7. No network calls or file modifications were made during the search.
