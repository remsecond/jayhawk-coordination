# Session-End Checklist

> Three steps before you wrap. Keeps the rehydration packet honest and the branch state current.

## 1. Update the rehydration packet's date + state line

Open `handoffs/2026-05-07_rehydration-packet.md`. Update:

- The `created:` / `last_updated:` front-matter date
- §0 if any new file landed on the branch
- §6 if any open question closed this session
- §8 if any new decision locked this session
- §9 if the "what still needs to happen" path changed

If nothing changed, skip. The packet only needs to reflect *landed* state, not in-flight chatter.

## 2. Commit anything pending

```
git status
git add -A
git commit -m "<verb-led message>"
```

Per the operating rule: anything that mattered this session should already be in files, not just in chat. If `git status` shows a clean tree, you're good.

## 3. Push

```
git push -u origin <branch-name>
```

The current active branch is `claude/review-openclaw-sessions-BNPso`. Don't push to `main` without instruction.

---

That's it. If a future Claude session opens this repo, `CLAUDE.md` points it at the rehydration packet, which now reflects where things actually stand. No archaeology required.
