---
name: remember-this
description: Store a distilled fact or decision in the memdb memory store for later recall across all repos. Use when the user says "remember this", "note this", "save this for later", "add this to memory", "remember that …", or otherwise asks to persist a fact/decision/recipe. Wraps `memdb add`.
---

# Remember this

Push a single distilled memory into the local `memdb` store so future sessions
(in any repo) can recall it via `recall-memory` / `memdb search`.

## When to invoke

Invoke when the user:

- Says "remember this", "note this", "save this", "add this to memory"
- Asks you to record a decision, recipe, or fact for later: "remember that we
  picked X", "save the fact that …", "note for next time"
- Wants a one-line distillation of *this* conversation's load-bearing outcome
  preserved

**Do NOT invoke** when:

- The user just wants you to *use* a past memory (that's `recall-memory`)
- The thing to remember is already obvious from the codebase or current context
- The user is asking to write a file, comment, or doc — that's a normal edit

## How to add

```bash
memdb add -- "<distilled text>"
```

- `--cwd` is auto-detected; the memory is stamped with the current repo so
  same-repo searches boost it later.
- The id is `"doc:" + sha256(text)` — adding the same text twice is idempotent.

## Distillation rules (these are what make the memory recallable)

The raw `memdb add` call is one line; **the value is in how you phrase it.**
Apply these rules before running the command:

1. **One fact per call.** If the user gives you three things to remember, make
   three `memdb add` calls. Don't pack a list into one memory — it dilutes
   embedding signal.
2. **Use the project's canonical terms.** If the codebase calls it `widget`,
   don't paraphrase as "UI component". Future queries with the user's real
   vocabulary should match.
3. **Lead with the decision or fact, not the journey.** "We picked subtree
   over filter-repo because rewriting history breaks existing clones" beats
   "we discussed the git strategy and considered several options".
4. **Include the why, briefly.** A decision without rationale is a stale rule.
   One clause of reasoning makes it usable when the tradeoff changes.
5. **≤ 1–2 sentences.** Anything longer should be a doc, not a memory. Dense
   embeddings reward focus.

## After the call

Report back concisely what was stored, e.g.:

> Stored (repo: `memdb`): *"We picked `modernc.org/sqlite` over
> `mattn/go-sqlite3` to avoid cgo — keeps the build pure-Go for cross-compiling."*

If `memdb add` fails (Ollama down, etc.), say so plainly and offer to retry.
Do not silently drop the request.

## Examples

User: "remember that we use subtree split, not git filter-repo, for extracting subdirs"
→ `memdb add -- "We use git subtree split (not git filter-repo) to extract subdirs into their own repos — keeps blame/history, avoids rewriting hashes that would break existing clones."`

User: "note that the auth flow uses env vars, not a secrets manager"
→ `memdb add -- "Auth uses DATABASE_URL / SECRET_KEY from env, not a secrets manager — keeps the deploy story one-line and avoids a vault dependency."`

User: "save the figma capture-widget recipe for later"
→ `memdb add -- "Figma capture-widget recipe: mount the iframe with allow-camera-permissions, post a capture request via postMessage, draw to a canvas, ship PNG via toBlob — works around Figma's sandbox."`
