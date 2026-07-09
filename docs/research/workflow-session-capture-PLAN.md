# PLAN — Capture Workflow / Subagent Session Logs (capture-only)

Implementation plan and progress tracker for fixing the missing workflow/subagent
session-log capture in the Claude Code provider. Companion to the diagnosis in
[`workflow-session-capture.md`](./workflow-session-capture.md).

- **Scope:** capture direction only (sandbox → host). Resume direction is **out of scope** for this change (see [Deferred](#deferred)).
- **Verified against:** `origin/main` @ `b4230ad`.
- **Changeset type:** `minor` (new capability, pre-1.0).
- **Status:** ✅ All steps complete (7/7) — ready to commit

---

## 1. Problem (one-paragraph context)

`captureToHost` in `makeClaudeSessionStorage` (`src/AgentProvider.ts:340`) copies only the
**main** session JSONL (`<sessionId>.jsonl`) from sandbox to host. When the agent spawns
**Workflow runs** or **subagents** (`Agent` tool), Claude Code writes their transcripts to
`~/.claude/projects/<encoded-cwd>/<sessionId>/subagents/agent-<agentId>.jsonl`. That directory
is never transferred, so those logs are lost when the container is torn down. A repo-wide search
for `subagent` in `src/**/*.ts` returns zero hits — confirmed unfixed.

On-disk layout:

```
~/.claude/projects/<encoded-cwd>/
  <sessionId>.jsonl                    ← main session  ✓ already captured
  <sessionId>/subagents/
    agent-<agentId>.jsonl              ← workflow / subagent  ✗ NOT captured
```

## 2. Current capture flow (what we're extending)

`captureToHost` today (`AgentProvider.ts:340–353`):

1. Compute **sandbox** source path — `claudeSandboxSessionPath(sandboxCwd, sessionId, sandboxProjectsDir)`.
2. Read it — `readSandboxFile(handle, sandboxPath, "claude-cap")` (`copyFileOut` → tempfile → `readFile`).
3. Rewrite cwd sandbox→host — `transferClaudeSession(jsonl, sandboxCwd, hostCwd)`.
4. Compute **host** dest path + `mkdir(dirname, { recursive: true })`.
5. `writeFile(hostPath, rewritten)`.

The fix adds: after step 5, enumerate `<sessionId>/subagents/agent-*.jsonl` in the sandbox and
run the same 5-step copy for each, writing under the host's `<sessionId>/subagents/` dir.

## 3. Key constraints discovered (why the plan looks the way it does)

- **A — `transferClaudeSession` is not crash-tolerant.** `rewriteSessionCwd` (`SessionStore.ts:100–126`)
  calls `JSON.parse` per line with **no try/catch** (unlike `transferPiSession`). A single truncated
  line in a crashed `agent-*.jsonl` would throw and abort the entire capture — including the main
  session. Must harden first.
- **B — subagent copies must be best-effort.** The main session is the contract; subagent logs are
  supplementary. One bad subagent file must not abort siblings or the main capture.
- **C — capture is directional & asymmetric.** Enumeration happens _in the sandbox_ (`handle.exec(find …)`),
  not on the host. Copy primitives are `readSandboxFile` + `writeFile`.
- **D — missing `subagents/` is the NORMAL case.** Plain sessions have no such dir. Unlike
  `locateCodexSandboxSession` (which throws on non-zero `find` exit), the enumerator must treat
  absent dir / empty output / non-zero exit as `[]`, never an error.
- **E — no logger in scope.** `makeClaudeSessionStorage` has no `display`/logger. Precedent for
  warnings is `console.error` (`SandboxFactory.ts`, `syncOut.ts`). "Log and continue" = `console.error` + swallow.

## 4. Path-builder convention

Follow the existing two-function split (`claudeSandboxSessionPath` / `claudeHostSessionPath`) rather
than a single `posix` flag, for consistency with the surrounding code.

---

## 5. Implementation steps (progress tracker)

> Update the checkbox and the **Status** line at the top as each step lands.

### Step 0 — Harden `rewriteSessionCwd` (constraint A)

- [x] Wrap the per-line `JSON.parse` in `rewriteSessionCwd` (`src/SessionStore.ts:100–126`) in try/catch; return the raw line on parse failure (mirror `transferPiSession:349–362`).
- [x] Unit test: a JSONL string with a malformed trailing line is returned with valid lines rewritten and the bad line preserved verbatim.
- **Files:** `src/SessionStore.ts`, `src/SessionStore.test.ts`

### Step 1 — Enumeration & path helpers (constraints C, D)

- [x] `claudeSubagentsDirInSandbox(cwd, sessionId, projectsDir)` → `<projectsDir>/<encoded-cwd>/<sessionId>/subagents` (POSIX join).
- [x] `claudeSubagentsDirOnHost(cwd, sessionId, projectsDir)` → host equivalent (`node:path` join).
- [x] `listClaudeSubagentSessionsInSandbox(cwd, sessionId, handle, sandboxProjectsDir): Promise<string[]>` — runs `find <dir> -type f -name 'agent-*.jsonl' 2>/dev/null`; **absent dir / empty / non-zero exit → `[]`**.
- **Files:** `src/SessionStore.ts`, `src/SessionStore.test.ts`

### Step 2 — Extract single-file copy primitive (constraint C)

- [x] Extract the `AgentProvider.ts:340–353` sequence into a reusable `copyClaudeSessionFile({ handle, sourcePath, fromCwd, toCwd, destPath, tag })` (read → `transferClaudeSession` → ensure dest dir → write).
- [x] Re-point the existing main-session capture through it (behavior unchanged).
- **Files:** `src/AgentProvider.ts`

### Step 3 — Rewire `captureToHost` (constraints B, E)

- [x] Main session copied via `copyClaudeSessionFile`; its failure stays **fatal** (current contract).
- [x] `listClaudeSubagentSessionsInSandbox(...)`; for each sandbox path compute host dest under `claudeSubagentsDirOnHost(hostCwd, …)` and copy (`posix.basename` of the sandbox path; `tag: "claude-cap-sub"`).
- [x] Each subagent copy wrapped in its own try/catch → `console.error` warning + continue. A subagent failure never aborts siblings or the main capture.
- **Files:** `src/AgentProvider.ts`

### Step 4 — Tests

- [x] `AgentProvider.test.ts`: reuse the existing real-fs `fsBindMountHandle()` (sandbox path == host path, so `find` runs against the staged temp dir).
  - [x] (a) no `subagents/` dir → only main session copied (host subagents dir asserted absent).
  - [x] (b) two `agent-*.jsonl` subagents copied to correct host paths, cwd rewritten sandbox→host on every line; a `notes.txt` sibling is NOT copied (proves the `-name 'agent-*.jsonl'` filter).
  - [x] (c) one subagent whose `copyFileOut` throws (I/O-layer failure, since Step 0 made content-rewrite crash-tolerant) → main + good sibling still captured; bad one absent; `console.error` emitted referencing the failing path.
- **Files:** `src/AgentProvider.test.ts`

### Step 5 — Changeset

- [x] Check `.changeset` for an existing duplicate first. (Only `config.json` + `README.md` present — no duplicate.)
- [x] Add a `minor` changeset; name from `package.json#name`. → `.changeset/capture-claude-subagent-logs.md` (`"@ai-hero/sandcastle": minor`).
- **Files:** `.changeset/*.md`

### Step 6 — Verify

- [x] `npm run typecheck` clean.
- [x] Affected test files pass (`SessionStore.test.ts`, `AgentProvider.test.ts`) — 257 passed, 0 failed.
- [x] Update [`workflow-session-capture.md`](./workflow-session-capture.md) §4/§6 to reflect the shipped shape and the resolved §5 caveat. (§4 retitled "Shipped Fix"; resume reframed as DEFERRED in §4/§5/§6; §5 caveat → "✅ Confirmed"; title/intro de-staled.)

---

## 6. Open verification item (carried from diagnosis §5)

⚠️ `transferClaudeSession` only rewrites top-level `cwd` and `session_meta.payload.cwd`. Whether
`agent-*.jsonl` transcripts actually carry those fields is **unconfirmed**. Applying the rewrite is a
safe no-op if they don't. During Step 4, inspect a real `agent-*.jsonl` fixture and record the finding
here:

- [x] Determined whether subagent logs carry rewritable `cwd` fields. **Finding: YES — confirmed.** Inspected a real `agent-*.jsonl` (58/58 lines parsed); **every line carries a top-level `cwd`** (`type: "user"` and `type: "assistant"` entries each have it). No line uses `session_meta.payload.cwd`. So the `transferClaudeSession` rewrite is genuinely load-bearing on subagent logs, not a no-op — tests assert the rewrite on every line.

## Deferred

- **Resume direction (host → sandbox).** Copying subagent logs _into_ the sandbox on resume is
  speculative — a normal `claude --resume <id>` loads the main session JSONL; it is unconfirmed that
  it reads `<id>/subagents/`. Out of scope here; revisit only with evidence Claude Code consumes that
  directory on resume.

## Follow-up backlog (discovered 2026-06-16 — handle in follow-up sessions)

Real `Workflow`-run logs were inspected on disk after this PR shipped. Full detail in
[`workflow-session-capture.md` §7](./workflow-session-capture.md#7-findings-from-inspecting-real-workflow-logs-2026-06-16--deferred-to-follow-up).
The shipped capture is correct for what it does (transcripts, with the existing cwd rewrite); the items
below are **deferred to follow-up sessions** and are _not_ regressions in this PR:

- [ ] **F1 — Preserve `workflows/wf_<runId>/` grouping (capture-side, low risk).** `captureToHost`
      flattens subagent logs to `basename` under the host `subagents/` dir, losing which Workflow run
      each transcript belonged to (no data loss observed — basenames were unique). Fix: write each file
      at its path _relative to_ the sandbox `subagents/` dir instead of `basename`. Path change only,
      no content rewrite.
- [ ] **F2 — Capture companion artifacts** (`agent-*.meta.json`, `journal.jsonl`, and the sibling
      `<sessionId>/workflows/` tree: run records + generated scripts). Currently never transferred; the
      `workflows/` tree is outside `subagents/`, so the enumerator never sees it. Needed for a faithful
      local mirror / any resume-side support (the Workflow tool resumes from the journal/run-record, not
      the transcripts). Tie to the resume work above, not to transcript capture.
- [ ] **F3 — New content rewrite for run-records/journals (gating F2).** The existing
      `transferClaudeSession` only rewrites an exact-match top-level/`session_meta.payload` `cwd` field
      and is a **no-op** on these files (no `cwd` key). A new rewrite must handle: (a) **cwd-as-prefix**
      (`<cwd>/subpath`, not a bare value); (b) the **encoded-cwd form** in `scriptPath`
      (`encodeProjectPath(from)→(to)`); (c) stay **cwd-scoped** — `args` carries a second, unrelated
      `repoRoot` that must NOT be rewritten. Transcripts keep using the existing rewrite unchanged.
- [ ] **(Confirmed not needed) Tool-path rewriting in transcripts.** Incidental sandbox paths in
      tool-call `command`/result fields are intentionally left unrewritten — matches shipped main-session
      behavior; tool history is a record, not re-executed. Do not "fix" this.

## Done log

- 2026-06-12 — Step 0: hardened `rewriteSessionCwd` with per-line try/catch (mirrors `transferPiSession`); added malformed-line preservation test. Accepted on first review.
- 2026-06-12 — Step 1: added `claudeSubagentsDirInSandbox` / `claudeSubagentsDirOnHost` / `listClaudeSubagentSessionsInSandbox` (absent-dir/empty/non-zero exit → `[]`, never throws) + tests. Accepted on first review.
- 2026-06-12 — Step 2: extracted the inline `captureToHost` copy sequence into module-level `copyClaudeSessionFile({ handle, sourcePath, fromCwd, toCwd, destPath, tag })`; re-pointed main-session capture through it (behavior identical, `tag: "claude-cap"`). Accepted on first review.
- 2026-06-12 — Step 3: rewired `captureToHost` — main copy stays fatal, then enumerates subagents via `listClaudeSubagentSessionsInSandbox` and copies each to `claudeSubagentsDirOnHost` (`posix.basename` dest, `tag: "claude-cap-sub"`), each in its own try/catch → `console.error` + continue. Accepted on first review.
- 2026-06-12 — Step 4: added 3 `AgentProvider.test.ts` tests (no-subagents, N-subagents-with-rewrite + `notes.txt` filter, corrupt-subagent-via-`copyFileOut`-throw) reusing `fsBindMountHandle()`; 216/216 pass. **Resolved §6**: subagent logs carry a top-level `cwd` on every line (58/58 in a real fixture) → rewrite is load-bearing, not a no-op. Accepted on first review.
- 2026-06-12 — Step 5: added `.changeset/capture-claude-subagent-logs.md` (`"@ai-hero/sandcastle": minor`; no duplicate existed). Accepted on first review.
- 2026-06-12 — Step 6: verified `npm run typecheck` clean + 257/257 tests pass (`SessionStore.test.ts` + `AgentProvider.test.ts`); updated `workflow-session-capture.md` (§4 → "Shipped Fix", resume DEFERRED across §4/§5/§6, §5 caveat → "✅ Confirmed", title/intro de-staled). Review pushed back once (§5 table row + title/intro still implied resume was in-scope); fixed on retry 1/3 and accepted.
