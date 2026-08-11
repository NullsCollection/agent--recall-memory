# AGENTS.md

## Status

**Pre-implementation.** This repo is a plan, not code. `PLAN.md` is the spec and the single source
of truth — read it before writing any Go. The decisions table in `PLAN.md` is **locked**; do not
relitigate vector store, embedding model, semantic unit, search scope, push trigger, or redaction
approach. If you disagree with a locked decision, surface it to the user rather than silently
picking differently. No `go.mod` exists yet — Phase 0 of `PLAN.md` lists the scaffold steps in order.

## Hard constraints (these will be violated by default)

- **One dependency only:** `modernc.org/sqlite` (pure Go, no cgo). Do not add another SQLite driver
  or reach for an ORM.
- **Four source files, hard limit.** `PLAN.md`: "Resist adding a fifth." Map all work to exactly one
  of `main.go` (CLI dispatch, `eval`, `stats`), `store.go` (schema, upsert, `loadAll`, `reindex`),
  `embed.go` (Ollama client, cosine, top-K), or `extract.go` (jsonl→units, redaction, short-turn
  filter). New harness sources go into `extract.go` as a function, not a new file or interface.
- **Brute-force cosine search.** No ANN index — corpus is ~hundreds of rows and sub-millisecond.
  sqlite-vec / ANN is a deferred item with an explicit trigger (~100k rows); do not add it
  speculatively.
- **Raw `text` is always stored next to `vector`.** Changing the embedding model is `memdb reindex`,
  never a schema migration. Do not drop or normalize the source text.

## External runtime

- **Ollama must be running** with `qwen3-embedding:0.6b` pulled. Verify with the curl in `PLAN.md`
  Phase 0. By design: search/embed fail loudly with an actionable message; `index --quiet` fails
  silently and the next session start retries. Do not "fix" the silent failure.
- **DB lives at `~/.claude/memdb.sqlite`** — outside the repo. `.gitignore` covers `memdb.exe` and
  `*.sqlite`; never commit a database.

## Working in this repo

- **Follow the phase gates.** Each phase in `PLAN.md` ends with a bold **Gate:** — do not start
  Phase N+1 until Phase N's gate passes. Phase 2 (`memdb eval` + `golden.txt`) is the one most
  tempting to skip and the most important: if recall is weak, fix extraction *before* building any
  integration on top. Do not ship Phase 3/4 on a store that hasn't passed `eval`.
- **Non-goals are load-bearing.** `PLAN.md` Phase 5 lists deferred items (auto-inject hook, OpenAI
  embeddings, LLM-distilled memories, ANN, per-workspace config, opencode/Codex indexing) **each
  with an explicit trigger condition**. Do not implement a deferred item unless its trigger has
  actually fired.
- **Redaction runs at ingest only, never on the read path.** The regex set lives in `extract.go`;
  surrounding prose must survive redaction (context is the valuable part). If you add a pattern,
  add a fixture test for it.
- **"Adding a source is another function, not an interface."** Don't design for N harnesses before
  the second one exists — see the Portability section of `PLAN.md`.

## Host vs. sandbox

`PLAN.md` targets a **Windows host** (the `SessionStart` hook spawns via `cmd /c start /b`; PATH is
`%USERPROFILE%\go\bin`). Go build, store, embed, and cosine logic are cross-platform, but the hook
command is Windows-only — do not rewrite it to a POSIX shell invocation when implementing Phase 4.
