---
name: recall-memory
description: Recall past decisions or prior work from the memdb memory store. Use when the user asks "remember when we…", "like last time", "how did we do X in the other repo", "did we already decide…", or otherwise references a past decision or a precedent. Searches indexed Claude Code sessions across all repos.
---

# Recall memory

Search past sessions and decisions indexed by `memdb` (a local SQLite store of
fused user/assistant turns). The point is cross-project recall — the answer to
"how did we solve this before" is often in another repo's session.

## When to invoke

Invoke when the user:

- References a past decision or prior work: "remember when we…", "did we
  already decide…", "what did we end up doing about X"
- Implies a precedent exists: "like last time", "the usual way", "our pattern for…"
- Asks about another repo or project: "how did we do X in the `<repo>` project"
- Asks about the rationale behind a current design that was decided earlier

**Do NOT invoke** when:

- The current conversation already answers the question
- It's general programming knowledge you already have
- The question is about the active repo's current state (use Read/Grep instead)
- The user is asking about something that happened *in this session*

## How to search

```bash
memdb search --top-k 5 -- "<query>"
```

- `--cwd` is auto-detected; same-repo hits get a small score boost.
- Phrase the query like the content you want — vocabulary match beats keyword
  tricks on dense embeddings. Quote the user's own words when possible.
- If the first query misses, try one rephrasing with different vocabulary before
  giving up. Two short searches beat one long one.

## Reading the results

Each hit prints `repo · kind · date · score`, then the indexed text (a fused
`User: … / Assistant: … / Tools: …` unit, or a `doc` for manual memories).
Higher score = better match.

- **Always state which repo a hit came from** when you use it — cross-project
  recall is the whole point, and the user needs to know whether to trust it for
  the current context.
- If a hit clearly answers, summarise what it says and cite the repo + date.
- If the top hit looks relevant but partial, surface what it actually says and
  name the gap — don't extrapolate beyond the indexed text.
- If nothing in the top-K clearly answers, say so plainly and move on. Don't
  fabricate from weak hits.

## Examples

User: "did we already figure out how to do the figma widget capture?"
→ `memdb search --top-k 5 -- "figma widget capture approach"`

User: "remember how the other repo handles admin auth?"
→ `memdb search --top-k 5 -- "admin authentication env password"`

User: "like we did before, split the monorepo"
→ `memdb search --top-k 5 -- "git subtree split monorepo"`
