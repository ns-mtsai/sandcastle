# Workflow Session Log Capture: Diagnosis and Shipped Fix

Research document compiled during investigation of the missing workflow session log transfer. Describes the current gap, the on-disk layout documented by Anthropic, and the shipped fix (capture) scoped to the Claude Code provider, with resume-side seeding deferred.

**Verified against `origin/main` at `b4230ad` (2026-06-11).** The implementation map in §3–4 reflects the actual `AgentSessionStorage` interface as it exists today, not the older `transfer()` surface earlier drafts described.

**Update (2026-06-16):** real `Workflow`-run logs were inspected on disk. The believed layout in §2 was incomplete — see **§7** for the verified nested layout, a transcript-flattening issue, uncaptured companion artifacts, and a content-rewrite gap. Those items are deferred to follow-up sessions.

## 1. The Problem

When Sandcastle runs a Claude Code agent inside a container, it captures the main session JSONL from the sandbox to the host after the agent finishes (`Orchestrator.ts:482–520`, via `provider.sessionStorage.captureToHost(...)`). However, if the agent invoked one or more **Workflow runs** (or spawned subagents via the `Agent` tool), those sub-sessions write their transcripts to a _different location_ that is never transferred. When the container is torn down, the workflow session logs are lost.

**Confirmed status (diagnosis):** was live on `origin/main` as of 2026-06-11. **Fixed — see §4.**

---

## 2. On-Disk Layout (Claude Code)

Source: official Claude Code documentation at `https://code.claude.com/docs/en/sub-agents` (retrieved 2026-06-09).

> "You can also ask Claude for the agent ID if you want to reference it explicitly, or find IDs in the transcript files at `~/.claude/projects/{project}/{sessionId}/subagents/`. Each transcript is stored as `agent-{agentId}.jsonl`."

> "Every run writes its script to a file under your session's directory in `~/.claude/projects/`."

The full layout:

```
~/.claude/projects/<encoded-cwd>/
  <sessionId>.jsonl                    ← main session  ✓ already captured
  <sessionId>/
    subagents/
      agent-<agentId>.jsonl            ← workflow / subagent session  ✗ NOT captured
      agent-<agentId>.jsonl
      ...
```

Key observations:

- The subdirectory is named after the **parent session ID**, creating an implicit parent→child link via directory structure.
- There is no explicit field in the JSONL content linking a child session back to its parent; the relationship is entirely positional.
- Sub-session file names use the `agent-<agentId>.jsonl` pattern (not `<agentId>.jsonl`), so a glob must account for the prefix.
- The `subagents/` directory only exists when the agent actually spawned at least one workflow or subagent run; it is absent for plain sessions.

---

## 3. Where the Fix Belongs

The fix belongs **inside the Claude Code provider's `AgentSessionStorage` implementation** (`makeClaudeSessionStorage` in `AgentProvider.ts`), not in the `Orchestrator`.

**Why not the Orchestrator?**
To enumerate the `subagents/` directory the Orchestrator would need to know: (a) the sandbox-side `~/.claude/projects/` path, (b) the `<encoded-cwd>` encoding scheme, and (c) the `<sessionId>/subagents/` layout. All three are Claude Code-specific. Pushing this knowledge into `Orchestrator.ts` would break the provider abstraction that ADR 0012 establishes. The Orchestrator already hands the provider everything it needs — `handle`, both cwds, and the `sessionId` — and treats "complete capture" as the provider's concern.

**Actual transfer surface (`origin/main`).** There is no single `transfer()` function and no separate "sandbox store" / "host store" objects. Each provider exposes one `AgentSessionStorage` (`AgentProvider.ts:231`) with two _directional_ methods:

```ts
captureToHost({ hostCwd, sandboxCwd, sessionId, handle }): Promise<void>    // sandbox → host
resumeIntoSandbox({ hostCwd, sandboxCwd, sessionId, handle }): Promise<void> // host → sandbox
```

Both receive the sandbox `handle` plus both cwds and the `sessionId`. The cwd rewrite itself is the pure helper `transferClaudeSession(jsonl, fromCwd, toCwd): string` (`SessionStore.ts:132`). File I/O lives in the `readSandboxFile` / `writeSandboxFile` helpers in `AgentProvider.ts` (`copyFileOut` to a tempfile / `copyFileIn` from a tempfile, with sandbox-side `mkdir -p`).

The Claude main-session paths already exist: `claudeSandboxSessionPath(cwd, id, projectsDir)` and `claudeHostSessionPath(cwd, id, projectsDir)` both build `<projectsDir>/<encoded-cwd>/<id>.jsonl` via `encodeProjectPath`. The subagents directory is therefore `<projectsDir>/<encoded-cwd>/<id>/subagents/` — derivable from the same primitives.

---

## 4. Shipped Fix

### Capture (`captureToHost`) — SHIPPED

The following was implemented in `src/AgentProvider.ts` and `src/SessionStore.ts`:

**New exports in `src/SessionStore.ts`:**

- `claudeSubagentsDirInSandbox(cwd, sessionId, projectsDir)` → `<projectsDir>/<encoded-cwd>/<sessionId>/subagents` (POSIX join).
- `claudeSubagentsDirOnHost(cwd, sessionId, projectsDir?)` → host equivalent (`node:path` join; defaults base to `~/.claude/projects`).
- `listClaudeSubagentSessionsInSandbox(cwd, sessionId, handle, sandboxProjectsDir): Promise<string[]>` — runs `find <dir> -type f -name 'agent-*.jsonl' 2>/dev/null` in the sandbox and returns absolute sandbox paths. Absent dir / empty stdout / non-zero exit all return `[]` — it never throws (unlike `locateCodexSandboxSession`, which throws on non-zero exit; a missing `subagents/` dir is the normal case for plain sessions).

Also hardened: `rewriteSessionCwd` (used by `transferClaudeSession`) now wraps the per-line `JSON.parse` in try/catch, returning malformed lines verbatim (mirrors `transferPiSession`). This makes capture crash-tolerant against partially-written/corrupt subagent logs.

**New helper in `src/AgentProvider.ts`:**

- `copyClaudeSessionFile({ handle, sourcePath, fromCwd, toCwd, destPath, tag })` — extracted single-file copy primitive: `readSandboxFile` → `transferClaudeSession(jsonl, fromCwd, toCwd)` → `mkdir(dirname(destPath), { recursive: true })` → `writeFile`. The main-session capture was re-pointed through it.

**`captureToHost` behavior:**

1. Copies the main `<sessionId>.jsonl` via `copyClaudeSessionFile` (`tag: "claude-cap"`) — failure remains **fatal** (the established contract).
2. Enumerates subagents via `listClaudeSubagentSessionsInSandbox(...)` (no-throw → `[]` when there is no `subagents/` dir, so the loop is a safe no-op for plain sessions).
3. Copies each subagent log to `join(claudeSubagentsDirOnHost(hostCwd, sessionId, hostProjectsDir), posix.basename(sourcePath))` via `copyClaudeSessionFile` (`tag: "claude-cap-sub"`), **each wrapped in its own try/catch → `console.error` + continue**. A single bad/unreadable subagent never aborts siblings or the already-completed main capture.

### Resume (`resumeIntoSandbox`) — DEFERRED

`resumeIntoSandbox` is **unchanged** — it still handles only the main session. Whether a `claude --resume <id>` invocation reads `<id>/subagents/` is unconfirmed; resume-side subagent seeding is out of scope pending evidence that it is needed. The same directory gap exists in the resume direction but was deliberately not fixed in this iteration.

### What did not change:

- `Orchestrator.ts` — calls `provider.sessionStorage.captureToHost(...)` / `resumeIntoSandbox(...)` unchanged.
- `AgentSessionStorage` interface — no `listSubagentSessions` method was added (would force Codex/Pi to implement a Claude-specific concept).
- `transferClaudeSession` pure rewrite logic — unchanged (the hardening is in `rewriteSessionCwd`, one layer below).
- Codex or Pi providers — this layout is Claude Code-specific.

---

## 5. Edge Cases & Caveats

| Case                                                                | Handling                                                                                                                                                                                                                                                   |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| No `subagents/` directory (plain session, no workflow runs)         | `find` returns nothing; transfer loop is a no-op.                                                                                                                                                                                                          |
| `subagents/` exists but is empty                                    | Same — no-op.                                                                                                                                                                                                                                              |
| Sub-session file has been partially written (agent crashed mid-run) | Copy whatever is there; partial JSONL is valid for replay/resume purposes.                                                                                                                                                                                 |
| Resume flow (host → sandbox)                                        | **Deferred** (see §4). The same directory gap exists host → sandbox, but resume-side subagent seeding was not shipped — `resumeIntoSandbox` still handles only the main session, pending confirmation that `claude --resume <id>` reads `<id>/subagents/`. |

**✅ Confirmed — cwd rewrite is load-bearing on sub-sessions.**
A real `agent-*.jsonl` was inspected: every line (58/58 in the sampled file) carried a top-level `cwd` field. Entries of `type: "user"` and `type: "assistant"` each carry it. No line used the `session_meta.payload.cwd` shape. The `transferClaudeSession` cwd-rewrite therefore rewrites subagent logs in exactly the same way it rewrites main-session logs — it is not a no-op.

---

## 6. Verification

Unit and integration-style tests were added in `src/AgentProvider.test.ts` and `src/SessionStore.test.ts`, reusing the existing real-fs `fsBindMountHandle()` fixture (sandbox path == host path, so the real `find` runs against a staged temp dir). Full suite: typecheck clean, all 257 tests in the two affected files pass.

**`SessionStore.test.ts`**

- Step 0: malformed-line preservation — `rewriteSessionCwd` returns corrupt/non-JSON lines verbatim (no throw).
- Step 1 unit tests for all three new helpers: `claudeSubagentsDirInSandbox`, `claudeSubagentsDirOnHost`, and `listClaudeSubagentSessionsInSandbox` (including absent-dir → `[]` and non-zero exit → `[]`).

**`AgentProvider.test.ts` — `captureToHost` integration cases**

- **(a)** No `subagents/` dir → only main session copied; host subagents dir asserted absent.
- **(b)** Two `agent-*.jsonl` subagents present → both copied to correct host paths; top-level `cwd` rewritten on every line; a `notes.txt` sibling in the same dir is **not** copied (confirms the `-name 'agent-*.jsonl'` filter).
- **(c)** One subagent whose `copyFileOut` throws (I/O-layer failure) → main session + good sibling still captured; bad subagent absent on host; `console.error` emitted.

**Item 4 — cwd field inspection: ✅ DONE.** See §5 resolved note. The rewrite is load-bearing.

**Item 5 — resume seeding: DEFERRED.** `resumeIntoSandbox` was not changed. Resume-side subagent seeding is out of scope pending confirmation that `claude --resume <id>` reads `<id>/subagents/`.

Existing tests to mirror for style: `SessionStore.test.ts` (pure transfer functions, path encoding) and the session-storage cases in `AgentProvider.test.ts`.

---

## 7. Findings from inspecting real Workflow logs (2026-06-16) — DEFERRED to follow-up

A session that ran four `Workflow` runs was inspected on the host
(`~/.claude/projects/<encoded-cwd>/<sessionId>/`). The real layout is **richer and more
nested** than §2 assumed, and surfaces issues the shipped capture does not yet handle.
**None of the below is fixed in the shipped PR** — all are deferred to follow-up sessions
(see the PLAN doc's "Follow-up backlog").

### 7.1 Verified on-disk layout (supersedes §2)

```
~/.claude/projects/<encoded-cwd>/
  <sessionId>.jsonl                                         ← main session            ✓ captured
  <sessionId>/
    subagents/
      agent-<agentId>.jsonl                                 ← plain Agent-tool subagent (flat)   ✓ captured
      workflows/
        wf_<runId>/
          agent-<agentId>.jsonl                             ← Workflow subagent transcript       ✓ captured (but flattened — 7.2)
          agent-<agentId>.meta.json                         ← {"agentType":"workflow-subagent"}  ✗ NOT captured
          journal.jsonl                                     ← per-run resume journal             ✗ NOT captured
    workflows/                                              ← SEPARATE sibling tree (outside subagents/)
      wf_<runId>.json                                       ← run record (status/tokens/result)  ✗ NOT captured
      scripts/
        <name>-wf_<runId>.js                                ← generated workflow script          ✗ NOT captured
```

Two subagent shapes coexist: plain `Agent`-tool subagents sit flat at `subagents/agent-*.jsonl`;
**Workflow** subagents nest two levels deeper at `subagents/workflows/wf_<runId>/agent-*.jsonl`.
The shipped recursive `find … -name 'agent-*.jsonl'` catches both (a `-maxdepth 1` variant would
miss every Workflow transcript — the bulk of them).

### 7.2 Transcript flattening (capture-side, low risk)

`captureToHost` writes each subagent log to `claudeSubagentsDirOnHost / posix.basename(sourcePath)`,
collapsing the `workflows/wf_<runId>/` grouping into one flat host `subagents/` dir. In the inspected
session all `agent-*.jsonl` basenames were globally unique → **no data loss**, but the host copy loses
_which Workflow run_ each transcript belonged to. **Proposed fix:** write each file to its path
_relative to_ the sandbox `subagents/` dir instead of `basename`, preserving `workflows/wf_<runId>/`.
This is a **path** change, not a content rewrite.

### 7.3 Uncaptured companion artifacts

`agent-*.meta.json`, `journal.jsonl`, and the entire `<sessionId>/workflows/` sibling tree (run records

- generated scripts) are **never transferred** (the latter is outside `subagents/`, so the enumerator
  never sees it). Transcripts are the bulk of the value (~17 MB vs ~0.5 MB for the `workflows/` tree), so
  transcript-only capture is defensible for forensics. But the Workflow tool **resumes from the journal /
  run-record**, not the transcripts — so a faithful local mirror (or any future resume-side support) needs
  these too. Tie this to the deferred resume work, not the capture PR.

### 7.4 Content-rewrite gap (the important one)

`transferClaudeSession` / `rewriteSessionCwd` rewrites a path **only** when a JSON entry has a key
literally named `cwd` whose value is an **exact whole-string match** for `fromCwd` (top-level or
`session_meta.payload.cwd`). No prefix/substring replacement; no other field. Consequences:

- **Transcripts (`agent-*.jsonl`) — leave as-is.** They carry the exact `cwd` field (rewritten,
  load-bearing) plus incidental sandbox paths inside tool-call `command`/result fields (e.g.
  `mkdir -p <cwd>/phoenix/…`) that the rewrite ignores. That is **correct and consistent**: the main
  session behaves identically (926 exact `cwd` fields rewritten, 183 tool-path prefixes left untouched),
  and tool history is a record, not re-executed. Adding tool-path rewriting would _diverge_ from shipped
  main-session behavior.
- **Run records / journals — the current rewrite is a NO-OP on them.** Neither file has a top-level
  `cwd` key (`has("cwd") === false`), so `transferClaudeSession` rewrites zero lines. They embed sandbox
  paths in forms it cannot touch:
  1. **cwd-as-prefix**, never bare — `result` / `args` / `scriptPath` hold `<cwd>/subpath` (one journal:
     0 exact matches, 46 prefix matches).
  2. **The encoded-cwd form** — `scriptPath` points into `~/.claude/projects/-Users-…-<cwd-encoded>/…`;
     a second encoding the rewrite has never handled (would need `encodeProjectPath(from)→(to)`).
  3. **A second, unrelated root** — `args` carries both `scanRoot` (= the cwd) **and** a separate
     `repoRoot` that is _not_ the cwd. A cwd-scoped rewrite correctly leaves `repoRoot` alone; a naïve
     "replace all `/Users/...` paths" approach would corrupt it. Any future rewrite **must stay
     cwd-scoped**, not blanket.
  - Generated `scripts/*.js` were **path-free** in the sample → no content rewrite needed; portable as-is.

**Bottom line:** capturing run-records/journals later requires a **new prefix + encoded-path,
cwd-scoped rewrite** — the existing exact-match `cwd` rewrite is insufficient (and silently no-ops) for
those artifacts. The shipped transcript capture needs no content-rewrite change.
