# Obsidian Blocks as Finer Granularity — Forwarded Take

> Verbatim capture from a Claude session conversation about Obsidian blocks, forwarded by Roberto on 2026-05-07. Lands here per the repo rule "if it matters, it lands here as a file + commit." The original conversation is not in the contract Claude's thread context; this file IS the rehydrated content.
>
> Not a contract. Not a decision. A substantive architectural take worth preserving so the next round of work can build on it. Pair with anything that ends up in `contracts/` if the ideas land formally.

---

You're right, and I missed the real point earlier. Obsidian blocks aren't just another query target—they solve a fundamental granularity problem in our operating model.

## The Real Insight

Our entire system works at file granularity. Lane cards are files. Sidecar YAMLs are files. Channel notes are files. Expert responses are files. Every unit of work, every piece of metadata, every tracked relationship is a file.

But knowledge, evidence, planning, and reasoning don't respect file boundaries. A single OFW message contains five distinct claims with different confidence levels. A single lane card contains multiple tasks in different states. A single expert response contains architectural judgments, design proposals, and caveats all jumbled together. When the smallest addressable unit is a file, all of that internal structure is invisible to the system.

Obsidian blocks solve this by making the smallest addressable unit a paragraph, not a file. And the ecosystem has quietly evolved to make blocks queryable, property-bearing, and relationship-aware.

## The Four Unique Data Qualities

### 1. Content-Addressable References (`^block-id`)

A block ID is a stable pointer to a specific paragraph, list item, or section—not just to the file that contains it. This means you can reference "the claim about camp fees in message 6798" rather than "message 6798." The reference survives even if the note is renamed, reorganized, or expanded with other content.

**Why this matters for us:** The Semantic Corpus Layer for OFW messages currently uses one sidecar YAML per message. But a single message can contain multiple distinct claims. With block references, each claim gets its own `^claim-id`, and cross-thread contradictions link claim-to-claim, not message-to-message.

### 2. Block-Level Properties (Inline Metadata)

The Block Properties plugin extends the `^block-id` syntax to carry inline key-value properties: `^my-block [status: draft, priority: high, assigned: Code]`. These aren't Dataview frontmatter fields—they attach to individual blocks within a note, not to the note as a whole.

The plugin also supports property values that contain links to other blocks: `^task-1 [blocked-by: ^task-2, depends-on: ^setup]`. This creates a bidirectional relationship graph at the block level. The plugin's graph view even visualizes these relationships, coloring nodes by status and rendering edges from typed links like `blocked-by` and `depends-on`.

### 3. Live Transclusion (`![[note#^block-id]]`)

Transclusion embeds a block's content directly into another note, and updates propagate automatically when the source changes. It's not a copy—it's a live window into the source block.

**Why this matters:** A Huddle could produce a directive block (`^huddle-directive-042`). That block gets transcluded into every lane card, spec, or channel note it affects. When the directive is refined, every transclusion updates. The directive lives in one place and appears everywhere relevant.

### 4. The MetadataCache Graph

Obsidian's internal MetadataCache already extracts and caches all structural elements—links, headings, tags, blocks, frontmatter—and maintains a graph of resolved links between files. This graph is queryable through the API and updates in real-time as files change.

This means block-level relationships are not just visual—they're computed data that scripts, Dataview queries, and plugins can consume. The infrastructure for block-level reasoning already exists in the vault engine.

## How This Complements Our System

| System Component | Current (File Granularity) | With Blocks |
|---|---|---|
| Semantic Corpus (OFW) | One sidecar YAML per message | Each claim within a message gets `^claim-id [source: christine, claim: financial_hardship, confidence: high]`. Cross-thread links connect specific claims, not entire messages. |
| Kanban Board | One lane card per file | Tasks within a planning note become individual blocks with `[status: blocked, assigned: Code]`. Dataview or the Block Properties plugin queries across blocks. |
| Huddle Output | Directive written as a new note | Directive produced as `^directive [huddle: HUD-001, status: candidate]`. Transcluded into every lane it affects. Refining the directive updates every transclusion. |
| Expert Responses | One candidate note per response | Key judgments, caveats, and recommendations are individual blocks with properties. The judgment block gets transcluded into the review; the caveat block links to the risk it identifies. |
| Channel Punch List | Agenda items as bullet points | Each item is a block with `[status: resolved, resolved-by: D-005]`. The Dependency Map can query all blocked items across the vault. |

## Is It Worth Exploring?

Yes, but start narrow. The danger with block-level granularity is fragmentation—too many addressable units, too much metadata to maintain, a system that's precise but unmanageable. The file is still the right default container for most work.

The highest-leverage first application is the Semantic Corpus Layer. You already have 17 messages with sidecar YAMLs. The next evolution is to tag individual claims within those messages as blocks, with cross-thread links between specific claims rather than between entire messages. That's where the legal value lives—counsel doesn't need to know "message 6798 contradicts message 7203," they need to know "the hardship claim in message 6798 is contradicted by the deposit acknowledgment in message 7203, and here are the exact paragraphs."

The Block Properties plugin is MIT-licensed, mature (v1.0 released January 2026), and works with your existing vault files without migration. You can install it, test it on the OFW corpus, and decide whether block granularity belongs in the system.

The files stay plain Markdown. The vault stays portable. The sidecar YAMLs still work. Blocks don't replace anything we've built. They add a finer resolution where it matters—and remain invisible where it doesn't.
