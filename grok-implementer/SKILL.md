---
name: grok-implementer
description: >
  Route hands-on implementation to Grok headless CLI — ONLY when the user
  explicitly asks for it (e.g. names Grok, or says "use grok-implementer").
  Never trigger automatically. When invoked, the orchestrator AI (Claude
  Code, Cursor, Codex-as-orchestrator, etc.) freezes a work-order spec, runs
  Grok to implement, then reviews the diff and re-runs proof. Skip when
  already inside a Grok session — never self-delegate.
---

# Grok Implementer

**Who runs this skill:** the *orchestrator* agent (Claude Code, Cursor Agent,
Codex when used as planner/reviewer, Windsurf, etc.).

**Who must not:** a Grok interactive or headless session acting as the
implementer — **never self-delegate** (infinite loop / double cost).

**When to use:** ONLY when the user explicitly requests it — e.g. they name
Grok, say "use grok-implementer", or ask to delegate the build to Grok. Do
**not** activate this skill on your own for ordinary implementation work;
absent an explicit request, the orchestrator implements normally.

**Rationale:** Long multi-file implement/fix/test loops are expensive on
metered orchestrator tokens. Grok headless is strong at mechanical coding
loops; the orchestrator wins at judgment, design, spec-writing, review, and
orchestration. So **Grok types; the orchestrator thinks and verifies**.

If both `codex-first` and `grok-implementer` apply:

1. Prefer the implementer the **user named**.
2. Else if the user is mid a Codex thread / plugin job → Codex.
3. Else if Grok auth is ready (`~/.grok/auth.json` or `XAI_API_KEY`) → Grok.
4. Else Codex if available; else implement tiny tasks yourself or ask the user.

## Route

**Delegate to Grok** (only once the user has explicitly asked for Grok):

- implementation from a frozen spec; refactors; mechanical migrations
- bug fixes with known repro; test writing; coverage fills
- CI fixes, dependency bumps, scripts/tooling
- bulk codebase exploration where raw reading ≫ the answer

**Keep on the orchestrator:**

- design, API design, architecture, naming, UX judgment
- tasks where writing the spec *is* the work (ambiguity = design)
- tiny edits (~<20 lines, single obvious change) — delegation overhead loses
- anything needing orchestrator-only tools: MCP (browser/computer-use),
  password managers, secrets UIs, product-specific integrations
- destructive/irreversible ops, releases, pushes, remote git mutations —
  orchestrator-side per project git rules
- **review of Grok output — never delegated, never skipped**

Mixed task: orchestrator designs first, freezes spec, delegates build-out.

Heuristic: prompt reads as a **work order** → delegate; writing it forces
design decisions → stay on the orchestrator.

## Binary / auth

- Binary: `grok` on PATH (typical install: `~/.grok/bin/grok`). Prefer
  `command grok` to bypass interactive shell wrappers. If missing:
  `export PATH="$HOME/.grok/bin:$PATH"`.
- Version floor: Grok Build ≥ 0.2.x (flags below verified on **0.2.93**).
- Auth: cached OAuth in `~/.grok/auth.json`, or `XAI_API_KEY`. On auth
  errors: stop and tell the user to run `grok login` (or set the key) — do
  not invent alternate auth flows.
- Headless triggers: `--prompt-file`, `-p`/`--single`, or `--prompt-json`.
  Do **not** pipe the prompt on stdin (Grok does not read prompt from stdin).

## Invoke profiles

Prompt via temp file, never inline quoting. Capture **JSON on stdout**
(there is no Codex-style `-o` flag).

### `implement` (default)

Single implementer agent. **No** `--check` (aligns with `codex-first`:
proof lives in the prompt; the orchestrator re-verifies after).

```bash
P=$(mktemp)
OUT="${TMPDIR:-/tmp}/grok-last.json"
ERR="${TMPDIR:-/tmp}/grok-last.err"
cat >"$P" <<'EOF'
## Goal
<one concrete outcome>

## Repo / paths
<absolute repo root + key files/dirs>

## Constraints
- Match existing style; surgical diffs only
- Don't touch <X>
- Do not git commit / push / force-push unless explicitly asked

## Non-goals
<what to leave alone>

## Proof expected
Run exactly: <test or build command>
Include exit code + relevant output in the final report.

## Output shape
1. Summary (what changed and why)
2. Files changed (paths)
3. Proof (command + exit code + tail of output)
4. Risks / follow-ups (only if real)
EOF

command grok \
  --prompt-file "$P" \
  --cwd <repo> \
  --always-approve \
  --output-format json \
  --effort high \
  --max-turns 50 \
  --no-subagents \
  --disable-web-search \
  >"$OUT" 2>"$ERR"
EC=$?
```

`--yolo` is an undocumented alias of `--always-approve`; prefer the
documented flag. Optional: `--no-auto-update` if your install accepts it
(hidden on some builds; not required when stderr is redirected).

### `verify-loop` (opt-in only)

Use when you want Grok to append a **self-verification loop** before the
final answer (extra turns; may spawn helpers).

```bash
command grok \
  --prompt-file "$P" \
  --cwd <repo> \
  --always-approve \
  --output-format json \
  --effort high \
  --max-turns 50 \
  --check \
  --disable-web-search \
  >"$OUT" 2>"$ERR"
```

**Do not combine `--check` with `--no-subagents`** — CLI error on 0.2.x:

```text
error: the argument '--check' cannot be used with '--no-subagents'
```

Grok claims after `--check` are still **advisory**; the orchestrator must
re-run proof.

### Effort / turns heuristic

| Task size | `--effort` | `--max-turns` |
|-----------|------------|---------------|
| Smoke / 1-file | `medium` | 15–25 |
| Normal implement (default) | `high` | 50 |
| Large migration | `high` or `xhigh` | 80+ (explicit) |

Levels: `none|minimal|low|medium|high|xhigh|max` (`--effort` ≡
`--reasoning-effort`).

### Parse result

Success JSON keys (verified 0.2.93): `text`, `sessionId`, `stopReason`,
`requestId`, optional `thought`.

Error object: `{"type":"error","message":"..."}`.

```bash
# always check error shape before reading .text
if [[ "$EC" -ne 0 ]] || jq -e '.type == "error"' "$OUT" >/dev/null 2>&1; then
  echo "Grok failed (exit=$EC)"; jq -r '.message // empty' "$OUT" 2>/dev/null
  tail -n 50 "$ERR"
  # do not silently re-implement the whole task on first failure —
  # fix auth/setup or tighten the spec and retry once
else
  # feed the orchestrator context only .text (+ sessionId); skip .thought
  jq -r '.text // empty' "$OUT"
  jq -r '.sessionId // empty' "$OUT" > "${TMPDIR:-/tmp}/grok-last.session"
fi
rm -f "$P"
```

### Flag reference

| Flag | Role |
|------|------|
| `--prompt-file` | Long work-order without shell quoting hell |
| `--cwd <repo>` | Lock workspace; Grok walks up to `.git` from here |
| `--always-approve` | Auto-approve tools (alias: `--yolo`). Required for unattended implement |
| `--output-format json` | Single JSON blob on stdout |
| `--effort high` | Reasoning depth |
| `--max-turns N` | Cap agent loops (min 1) |
| `--no-subagents` | One implementer agent (**incompatible with `--check`**) |
| `--disable-web-search` | Pure code tasks; drop if docs/API lookup needed |
| `--check` | Opt-in self-verify loop (**incompatible with `--no-subagents`**) |

Optional tighteners (sensitive repos):

```bash
--rules "Only modify paths under src/ and test/. Never edit .env, secrets, or git config. Never git push."
--deny "Bash(git push*)" --deny "Bash(git reset*)"
```

Avoid blanket `--deny "Bash(rm*)"` unless the repo is high-risk — it blocks
legitimate cleanup.

Do **not** use `-s/--session-id` to continue work — that only **creates** a
new UUID session and errors if it already exists. Resume with `-r` /
`--resume` or `-c` / `--continue` only.

Notes:

- stderr is logs/noise; keep it out of the orchestrator transcript
  (`2>"$ERR"`). Open `$ERR` only to debug failures.
- long runs: background the shell job, read `$OUT` on exit; don't kill quiet
  runs under ~30 min without cause.
- parallel independent tasks OK: separate `--cwd`, separate `$OUT` / session
  files.
- monorepo: point `--cwd` at the **subproject**, not a huge parent.
- isolation (optional): `--worktree [name]` so Grok edits in a worktree;
  orchestrator reviews/merges afterward.

## Follow-up (resume)

Cheaper than fresh runs; keeps Grok context. Prefer explicit session id:

```bash
P2=$(mktemp)
OUT="${TMPDIR:-/tmp}/grok-last.json"
ERR="${TMPDIR:-/tmp}/grok-last.err"
SID=$(cat "${TMPDIR:-/tmp}/grok-last.session")
cat >"$P2" <<'EOF'
## Follow-up
<what failed / what to change>

## Proof expected
<same or updated test command>
EOF

command grok \
  --prompt-file "$P2" \
  --cwd <repo> \
  --resume "$SID" \
  --always-approve \
  --output-format json \
  --effort high \
  --max-turns 40 \
  --no-subagents \
  >"$OUT" 2>"$ERR"
# parse same as Invoke
rm -f "$P2"
```

If you lost the session id but stayed in the same repo cwd:

```bash
command grok --prompt-file "$P2" --cwd <repo> --continue \
  --always-approve --output-format json ...
# --continue / -c = most recent session for that working directory
# --resume with no id also resumes the most recent session
```

Fresh thread (abandon prior context): omit `--resume`/`--continue` and run
a new `implement` invoke.

## Prompt contract

Grok starts with **zero orchestrator session context**. Every prompt must
include:

1. **Goal** — concrete outcome
2. **Repo + paths** — absolute root and the files that matter
3. **Constraints** — don't-touch list, style, no push/commit unless asked
4. **Non-goals** — prevent scope creep
5. **Proof expected** — exact command the **orchestrator** will re-run
6. **Output shape** — files changed + proof output

Spec quality decides success. Ambiguous prompts → design on the
orchestrator first, then delegate.

## Verify (orchestrator, always)

- `git status -sb` + read the full diff; judge like a contributor PR
- re-run focused tests yourself (or demand proof output and re-run the same
  command); Grok claims — including after `--check` — are **advisory**
- iterate via `--resume` with the saved `sessionId`
- after **2** failed rounds, take over and implement directly
- if Grok never started (auth/binary missing), report that and stop — do
  not silently re-implement the whole task unless the user asks
- normal closeout still applies: review before ship

### Failure ladder

| Failure | Action |
|---------|--------|
| Auth / binary missing | Stop; tell user `grok login` or install/PATH — do not re-implement |
| Flag / parse error | Fix the invoke (skill/CLI mismatch); re-run |
| Task fail (tests red / wrong diff) | `--resume` once with a tighter follow-up spec |
| 2nd task fail | Orchestrator takes over |
| Partial write + dirty tree | Inspect `git status` + diff before resume or takeover |

## Economics

Win = generation + exploration + tool-loop tokens moved to Grok; the
orchestrator spends only on **spec + diff review**. Don't ping-pong trivia
through delegation; don't re-read what Grok already summarized in `.text`
(and don't dump `.thought` into context).
