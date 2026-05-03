# Tab GitHub write directive (optional)

Tab is an optional “hands” surface because it can use the operator’s authenticated browser session.
This does **not** change the auth rule: **only the operator pushes to main**.

Use this directive when an actor has content that should land in `jayhawk-coordination`, but the actor cannot push (no GitHub auth) and you want a consistent, low-error UI workflow.

## Paste-ready directive template

Land this file in the `jayhawk-coordination` repo via the GitHub web UI.

Repo: https://github.com/remsecond/jayhawk-coordination
Branch: <branch name>
File path: <path inside the repo>
Commit message: <verbatim commit message>
Open PR: <yes/no, against which base>

File content (paste verbatim into the web editor):

<begin>
[full markdown body]
<end>

Confirm when done with the commit URL or the PR URL.

## Limitations

- Web editor is best for small Markdown changes.
- One file per commit is the default; multi-file changes are multiple commits (or “Upload files”).
- Renames and large diffs are clunky in the web editor.

## Verification step

After Tab reports success, another actor can verify by fetching the raw URL or checking the PR/commit link.
