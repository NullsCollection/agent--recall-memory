# memdb — agent memory for Claude Code (Go)

A standalone Go CLI that owns a SQLite memory store. Claude Code talks to it through thin glue
(a skill now, a hook later). Ported from `agent-memory-system.md`, adapted for Go + Claude Code.

## Decisions (locked during grilling)

| #   | Decision      | Choice                                                         | Why                                                                                                                          |
| --- | ------------- | -------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 1   | Vector store  | SQLite + brute-force cosine in Go                              | LanceDB has no Go binding. Corpus is ~457 rows — linear scan is sub-millisecond.                                             |
| 2   | Embeddings    | Ollama `qwen3-embedding:0.6b` (1024-dim, local)                | Free backfill, no network on the read path, coding-aware. Swap to OpenAI later via `reindex`.                                |
| 3   | Semantic unit | Fused turn + short-turn filter                                 | `User: … / Assistant: … / Tools: …`. Short/approval turns borrow the preceding assistant message.                            |
| 4   | Search scope  | Global, same-repo score boost, repo shown on every hit         | Cross-project recall is the payoff; the boost keeps local hits on top.                                                       |
| 5   | Push trigger  | `SessionStart` hook, fire-and-forget, mtime guard              | No scheduler to maintain. First backfill is manual and watched.                                                              |
| 6   | Pull          | Skill first (model decides), auto-inject hook deferred         | Skill gives a clean signal on whether recall is worth anything, and produces the distances the hook's threshold needs.       |
| 7   | Secrets       | Regex redact-in-place at ingest                                | Pull path can leak a repo-A secret into a repo-B session. Redact, don't drop — the surrounding context is the valuable part. |
| 8   | Validation    | Golden-query file + `memdb eval`                               | Turns every tuning question into a number instead of a vibe.                                                                 |
| 9   | Location      | `Repo\memdb\`, module `github.com/raffy/memdb`, binary `memdb` | `go install` puts it on PATH. DB at `~/.claude/memdb.sqlite`.                                                                |

**Non-goals for v1:** LLM-distilled memories, ANN indexing, multi-user/server deployment,
per-workspace config files, auto-inject on every turn. Each has a named trigger below.

---

## Architecture

```
~/.claude/projects/**/*.jsonl ──► memdb index ──┐
                                                 ├──► ~/.claude/memdb.sqlite
               "remember this" ──► memdb add  ────┘             │
                                                               ▼
                             recall skill ──► memdb search ──► top-K hits
```

One binary, single writer. Every row keeps its raw `text` next to its `vector`, so changing the
embedding model is `memdb reindex`, not a migration.

### Schema

```sql
CREATE TABLE memories (
  id     TEXT PRIMARY KEY,  -- sha256(sessionID + entryUUID) | "doc:" + sha256(text)
  text   TEXT NOT NULL,     -- the raw fused unit, post-redaction
  vector BLOB,              -- []float32, little-endian
  dim    INTEGER,
  model  TEXT NOT NULL,     -- embedding model that produced `vector`
  cwd    TEXT,
  repo   TEXT,              -- last path segment of cwd, for the boost + display
  kind   TEXT,              -- "turn" | "doc"
  ts     INTEGER            -- unix seconds
);
CREATE TABLE meta (k TEXT PRIMARY KEY, v TEXT);  -- last_indexed
```

Deterministic `id` makes re-indexing an idempotent upsert. 457 rows x 1024 dims x 4 bytes ≈ 1.8 MB.

### Files

Four Go files. Resist adding a fifth.

- `main.go` — flag parsing, subcommand dispatch, `eval`, `stats`
- `store.go` — schema, upsert, load-all, `reindex`
- `embed.go` — Ollama HTTP client, cosine, scoring + top-K
- `extract.go` — jsonl → units, redaction, short-turn filter

### CLI surface

```
memdb index [--force] [--quiet]      walk sessions, embed, upsert (idempotent)
memdb add [--cwd .] -- <text>        push a manual/document memory
memdb search [--top-k 5] [--cwd .] [--json] -- <query>
memdb reindex                        re-embed all stored text on the current model
memdb stats                          row counts by repo / kind / model
memdb eval [--golden golden.txt]     run golden queries, print hit/miss + rank
```

---

## Phase 0 — Scaffold and prerequisites

- `mkdir Repo\memdb` and `git init`
- `go mod init github.com/raffy/memdb`
- `go get modernc.org/sqlite` (pure Go, no cgo — the only dependency)
- `ollama pull qwen3-embedding:0.6b` (~640 MB)
- Verify Ollama answers: `curl http://localhost:11434/api/embed -d '{"model":"qwen3-embedding:0.6b","input":"hello"}'` returns a 1024-length array
- Confirm `%USERPROFILE%\go\bin` is on PATH
- `.gitignore`: `memdb.exe`, `*.sqlite`

---

## Phase 1 — The engine

Nothing else works until `index` then `search` works from a shell. No Claude Code integration in
this phase.

### 1a — Store

- `store.go`: open DB at `~/.claude/memdb.sqlite`, create schema if absent
- `upsert(row)` — `INSERT … ON CONFLICT(id) DO UPDATE`
- `loadAll(model)` — read every row matching the current embedding model into memory
- float32 ⇄ BLOB helpers (little-endian, `encoding/binary`)
- `meta` get/set for `last_indexed`

### 1b — Embedding

- `embed.go`: POST to `/api/embed`, parse `embeddings[0]`
- Fail loudly with an actionable message if Ollama is down or the model isn't pulled
- Batch where the API allows it; otherwise a small worker pool (4 goroutines) for backfill
- Hard-code model name + dim as consts; both are stamped on every row

### 1c — Extraction

- Walk `~/.claude/projects/*/*.jsonl`
- Skip files with mtime **newer than 24h** (unsettled sessions)
- Skip files with mtime **older than `last_indexed`** unless `--force`
- Parse each line; keep `type == "user"` entries with real text (string content, or `text` blocks — **not** `tool_result`)
- Drop harness noise: text starting with `<command-name>`, `<local-command`, `<system-reminder>`, `Caveat:`
- For each kept user turn, scan forward to the next user turn and collect: assistant `text` blocks + `tool_use` **names only** (never tool output)
- Fuse into `User: …\nAssistant: …\nTools: Read, Edit, Bash`
- Short-turn rule: if the user text is under ~40 chars or is a bare approval (`ok`, `yes`, `A`, `continue`, `go`, `sure`), prepend the **preceding** assistant message so the decision survives. If there is no preceding assistant message, drop the unit.
- Truncate each unit to ~4000 chars
- Derive `cwd` from the project dir name; `repo` = last segment
- `id = sha256(sessionID + entryUUID)`

### 1d — Redaction (runs before embedding, never on the read path)

- Regex replace → `[REDACTED]`: `sk-[A-Za-z0-9]{20,}`, `ghp_\w{36}`, `github_pat_\w+`, `AKIA[0-9A-Z]{16}`, `xox[baprs]-[\w-]+`, `(postgres|mysql|mongodb)(\+srv)?://[^\s]*:[^\s]*@[^\s]+`, `Bearer\s+[A-Za-z0-9._-]{20,}`, `-----BEGIN [A-Z ]*PRIVATE KEY-----[\s\S]*?-----END [A-Z ]*PRIVATE KEY-----`
- Unit test: a fixture string containing each pattern comes out fully redacted with surrounding prose intact

### 1e — Search

- Embed the query
- Cosine over all in-memory vectors
- Same-repo boost: `score *= 1.15` when `row.repo == cwd repo` — a named const, tuned in Phase 2
- Top-K, print `repo · kind · date · score` then the text
- `--json` flag for machine consumption

### 1f — Remaining commands

- `stats` — counts by repo, kind, model; `last_indexed`
- `add` — same insert path, `kind = "doc"`, `id = "doc:" + sha256(text)`
- `reindex` — re-embed every row's stored text on the current model; skip rows already stamped with it unless `--force`

---

## Phase 2 — Prove it, then tune it

The point of this phase is a number. Skipping it means every later choice is a guess.

- `memdb eval` — parses `query ||| expected-substring`, sweeps `--top-k` / `--repo-boost` / `--query-prefix` in-memory (no rebuild), prints HIT/MISS+rank, summary line `X/Y hit (Z%), mean rank N`, and the raw-cosine min/median/max of good hits as the Phase 5 threshold signal. `--short-turn` is a flag on `index` (re-extracts + re-embeds).
- `golden.txt` — scaffold with known queries; fill in real expected substrings + more from sessions
- Skim 5–10 indexed sessions; find ~10 things you genuinely want recallable
- Fill in `golden.txt` with real expected substrings — each substring grepped against the store first to confirm it exists and only matches rows about that decision
- Record the baseline score

Then tune, re-running `eval` after each change:

- Try `top-k` 3 / 5 / 10
- Try the repo boost at 1.0 (off) / 1.15 / 1.3
- Try the short-turn threshold at 20 / 40 / 80 chars
- Try `qwen3-embedding`'s retrieval instruction prefix on queries
- Note the min/median/max distance of _good_ hits — Phase 5's auto-inject threshold should sit just below the min

---

## Phase 3 — Pull: the recall skill

Do this before push automation — it's what makes the store useful, and the manual backfill from
Phase 1 already has data in it.

- Create `~/.claude/skills/recall-memory/SKILL.md` (source written in repo at `skills/recall-memory/SKILL.md`; copy or symlink to `~/.claude/skills/recall-memory/` on the host)
- Description written so it triggers on: "remember when we…", "like last time", "how did we do X in the other repo", "did we already decide…"
- Body instructs: run `memdb search --top-k 5 -- "<a query phrased like the thing you're looking for>"`, prefer low-distance hits, always state which repo a hit came from, don't call it for general knowledge or when context already answers
- Add `Bash(memdb search:*)` to `permissions.allow` in `~/.claude/settings.json`
- Test in a fresh session in an unrelated repo: ask about a past decision, confirm the skill fires and the hit is right

**Install on the host** (Git Bash):

```bash
mkdir -p ~/.claude/skills/recall-memory
cp ~/Desktop/Repo/memdb/skills/recall-memory/SKILL.md ~/.claude/skills/recall-memory/SKILL.md
# or, to track edits in-repo:  ln -s ~/Desktop/Repo/memdb/skills/recall-memory ~/.claude/skills/recall-memory
```

Then add to `~/.claude/settings.json` under `permissions.allow` (create the key/array if absent):

```jsonc
{
  "permissions": {
    "allow": ["Bash(memdb search:*)"],
  },
}
```

Then test in a **fresh** Claude Code session in an unrelated repo — ask "remember when we …?" about a real past decision, and confirm: (a) the skill fires with no manual tool invocation, (b) no permission prompt appears, (c) the hit is right.

### 3b — opencode as a second reader (optional, ~2 minutes)

Memories come from Claude Code; opencode just reads them. Costs nothing but a line of prose.

- Add to `~/.config/opencode/AGENTS.md` — the four trigger phrases, when _not_ to search, and the say-which-repo rule. `memdb` is on the PATH opencode inherits, so no further wiring.
- Test from an opencode session in an unrelated repo
- _Upgrade path:_ if the agent keeps forgetting the tool exists, promote it to a plugin tool —
  ~15 lines of JS in `~/.config/opencode/plugins/memdb.js` shelling out to the binary.
  You already have a working plugin there (`herdr-agent-state.js`) to copy the shape from.

---

## Phase 4 — Push automation

- `--quiet` on `index`: no stdout on success (SessionStart output is injected as context — noise here pollutes every conversation)
- Confirm the no-op path is fast: `memdb index` with nothing new should be ~69 `stat` calls and exit in milliseconds
- Add the `SessionStart` hook to `~/.claude/settings.json`, detached so a slow run never blocks the prompt: `cmd /c start /b memdb index --quiet`
- Verify it does not block session start and produces no visible output
- Verify a session from 2 days ago gets picked up automatically on the next start
- Verify today's live session is **not** indexed (24h rule)
- Create `~/.claude/skills/remember-this/SKILL.md` wrapping `memdb add` — distilled fact per call, project's canonical terms, report success back (source at `skills/remember-this/SKILL.md` in the repo; install by copy to `~/.claude/skills/remember-this/`)
- Add `Bash(memdb add:*)` to `permissions.allow`

**Install on host** (Git Bash):

```bash
# 1. remember-this skill (same shape as recall-memory install)
mkdir -p ~/.claude/skills/remember-this
cp ~/Desktop/Repo/memdb/skills/remember-this/SKILL.md ~/.claude/skills/remember-this/SKILL.md
```

Then add both permission + hook to `~/.claude/settings.json` (merge with any existing keys):

```jsonc
{
  "permissions": {
    "allow": ["Bash(memdb search:*)", "Bash(memdb add:*)"],
  },
  "hooks": {
    "SessionStart": [
      { "type": "command", "command": "cmd /c start /b memdb index --quiet" },
    ],
  },
}
```

`cmd /c start /b …` is the detached-spawn form — `start /b` returns immediately (no new window), `cmd /c` exits as soon as it's launched. So Claude Code never waits on `memdb` and no output is injected into the session. If Ollama is down the hook fails silently under `--quiet`; the next session start retries.

**Verify (3 quick checks on host):**

1. **No block / no output** — open a fresh Claude Code session; the prompt appears immediately and no `indexed N units` line shows up in the conversation. (`memdb stats` afterward should show `last_indexed` updated to ~now if there was anything new, or unchanged if there wasn't.)
2. **Auto-pickup** — make a tiny edit to a 2+ day old session jsonl (e.g. `touch` it), start a new session, confirm `memdb stats`'s row count grows on the next start.
3. **24h exclude** — confirm today's live session is **not** in `memdb stats`'s by-repo counts (its dir should be absent or its row count unchanged from the backfill).

---

## Phase 5 — Deferred (build only on the named trigger)

- **Auto-inject via `UserPromptSubmit` hook.** _Trigger:_ Phase 3 works but you keep noticing the model failed to search when it should have. Requires the distance threshold measured in Phase 2 — inject nothing rather than inject junk. Re-run `eval` after, and be honest about whether answers got better or just noisier.
- **Swap to OpenAI embeddings.** _Trigger:_ eval plateaus and you suspect the model. It's `reindex` + a const, not a migration — that's the whole reason raw text is stored.
- **LLM-distilled memories.** _Trigger:_ eval stays weak after Phase 2 tuning, i.e. the fused-turn unit is genuinely the ceiling.
- **sqlite-vec / ANN index.** _Trigger:_ `memdb stats` passes ~100k rows and search is measurably slow. At current growth that's years away.
- **Per-workspace config.** _Trigger:_ you actually want different behaviour in two repos. Not before.
- **Index opencode history.** _Trigger:_ you start using opencode for real work. ~20 lines:
  one SQL query, no parsing. **Open the DB read-only** (`file:…/opencode.db?mode=ro`) — it's
  WAL and opencode may be running.
  ```sql
  SELECT m.session_id, m.id, m.time_created, s.directory,
         json_extract(p.data,'$.text')
  FROM part p JOIN message m ON m.id = p.message_id
              JOIN session s ON s.id = m.session_id
  WHERE json_extract(p.data,'$.type') = 'text'
  ```
- **Index Codex history.** _Trigger:_ you want Codex's past work recallable. `~/.codex/sessions/`
  has rollout `.jsonl` files — same line-typed shape as Claude Code, and `session_meta`
  carries `cwd` directly, so it's slightly easier. ~40 lines, a second extract function.

---

## Portability

The store is the protocol. Only extraction is harness-specific.

| Component    | Portable       | Notes                                                |
| ------------ | -------------- | ---------------------------------------------------- |
| `store.go`   | ✅ 100%        |                                                      |
| `embed.go`   | ✅ 100%        |                                                      |
| `main.go`    | ✅ 100%        |                                                      |
| `extract.go` | ❌ per-harness | one function per source, writing into the same table |

| Harness         | Push (read its history)               | Pull (use memories)                                     |
| --------------- | ------------------------------------- | ------------------------------------------------------- |
| **Claude Code** | `SessionStart` hook — Phase 4         | skill — Phase 3                                         |
| **opencode**    | SQL query — deferred, low value today | `AGENTS.md` line — Phase 3b, free                       |
| **Codex**       | jsonl walk — deferred                 | `~/.codex/skills/` exists; likely same shape as Phase 3 |

Adding a source is **another function, not an interface**. One implementation doesn't need an
abstraction; build the second one before designing for N.

The asymmetry works out: memories are produced by Claude Code and can be consumed by anything
that can run a shell command.

---

## Risks

| Risk                                                      | Mitigation                                                                                                              |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| 457 rows is too small to be useful                        | Phase 2's gate catches this before any integration is built on top                                                      |
| Ollama not running when the hook fires                    | `index` fails silently under `--quiet`; the next session start retries. Search fails loudly with an actionable message. |
| Redaction regexes miss a secret                           | Accepted. 15 lines that catch the realistic cases; the DB is local-only either way.                                     |
| SessionStart hook slows every launch                      | mtime guard + detached spawn; verified explicitly in Phase 4                                                            |
| Extraction breaks when Claude Code changes its log format | `stats` shows row count flatlining. Format is versioned per-line by `type`; unknown types are skipped, not fatal.       |
