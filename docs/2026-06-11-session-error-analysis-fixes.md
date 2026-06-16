# Session Error Analysis & Fix Plan — 2026-06-11 (Plan A)

Implementation plan for fixing issues discovered by analysing every Pi session under
`~/.pi/agent/sessions/` that used the `pi-interactive-subagents` extension.

This is **Plan A** of three. Companion plans, executed in later sessions in this order:

- **Plan B** — [Subagent Handoff Improvements](./2026-06-11-subagent-handoff-improvements.md)
  (structured `subagent_done` payload, summary extraction, write allowlists; from the
  June-4 handoff RCA)
- **Plan C** — [Auto-Resume on Transient Provider Drops](./2026-06-11-auto-resume-transient-drops.md)
  (formerly Fix 5 of this plan; pulled out as standalone)

**Scope rule (karpathy):** every changed line must trace to a listed fix. No adjacent
refactoring, no drive-by cleanups in the 2,400-line `index.ts`, no speculative handling of
states that can't occur.

**Scope of analysis:** 721 session JSONL files, 425 `subagent` tool calls across 85 sessions,
plus 135 `subagent_done`, 8 `subagent_resume`, 6 `subagents_list`, 2 `subagent_interrupt`,
1 `caller_ping` calls. Runtime: pi 0.79.1 (`@earendil-works/pi-coding-agent`). Extension loaded
globally as local-path package `../../devel/pi-interactive-subagents` from
`~/.pi/agent/settings.json`.

---

## Findings

| # | Issue | Evidence | Disposition |
|---|-------|----------|-------------|
| 1 | Child pi crashes at startup (exit 1) when two extension copies both register `subagent*` tools | ≥7 planner spawns failed (May 17, 18, Jun 4). Captured child stderr: `Error: Failed to load extension ".../git/github.com/Mathuv/pi-interactive-subagents/...": Tool "subagent" conflicts with /Users/mathu/devel/pi-interactive-subagents/...` (×4 tools). pi 0.79.1 `dist/main.js:600` exits 1 on any extension error diagnostic. Scouts survived only because `spawning: false` ⇒ `PI_DENY_TOOLS` suppressed registration in both copies. | **Fix 1** (resilience) |
| 2 | Startup death reported as bare `Sub-agent exited with code 1` plus `Session:`/`Resume:` lines pointing at a session file that was never created | June 4 session burned 4 failed planner retries chasing a wrong model theory; the real stderr was only recovered by manually re-running the launch script with `bash -x` | **Fix 2** |
| 3 | Agent discovery ignores the `cwd` tool param; unknown `agent:` names silently launch with no defaults | June 4: `.pi/agents/planner-nosub.md` created in the target repo was invisible to the parent (`loadAgentDefaults` only checks parent `process.cwd()`); the spawn proceeded with null defs (no model, no deny-tools, no system prompt) instead of erroring | **Fix 3** |
| 4 | `skillPaths is not iterable` tool errors (3×, May 22) | Mid-development bug in the bootstrap feature, fixed the same day by `c12be35` (current code passes `skillPaths: []`, matching pi 0.79.1 `LoadSkillsOptions`). Root enabler: devDeps pin `@mariozechner/pi-coding-agent ^0.65.0` while the runtime is `@earendil-works/pi-coding-agent` 0.79.1 | Fixed; **Maintenance** for the drift |
| 5 | Reviewer subagents failed in 4–5 s: `The 'gpt-5.3-codex' model is not supported when using Codex with a ChatGPT account` (3×, Jun 2–3) | Provider 400 surfaced via the provider-error path; cost a wasted spawn each time. `~/.pi/agent/agents.json` already remediated (reviewer → `openai-codex/gpt-5.4`) | **Fix 4** (preflight) |
| 6 | Transient provider drops kill subagents outright: `WebSocket closed 1000` / `1012` / `WebSocket error` (4×, May 24 – Jun 10) | Correctly surfaced since `05be68b`, but each one requires a manual re-spawn/resume by the orchestrator | **Plan C** (auto-resume, standalone) |
| 7 | User-cancelled subagents render as `failed (exit code 1)` with summary `Subagent cancelled.` (3×) | Cancellation is not a failure; current presentation is misleading in the result box | **Fix 6a** |
| 8 | `loadStatusConfig()` throws at module top level if both `config.json` and `config.json.example` are missing | Code inspection (`status.ts`). A packaging/installation that drops the example file would make the extension import throw → pi exits 1 at startup | **Fix 6b** |
| 9 | Stale/environmental one-offs: SIGSEGV exit 139 (May 27, caused by the since-uninstalled symbol-autocomplete extension), `Tool subagent not found` (Apr 29, pre-install), `not_found: Surface not found` (Apr 3, old cmux) | Root causes gone | No action |
| 10 | Subagent handoff weaknesses (zero-param `subagent_done`, last-assistant-text summaries hiding artifacts, ignored `output:` frontmatter, scout/reviewer write-tool mismatch, orchestrator wait discipline) | June-4 21:12 RCA session (`2026-06-04T21-12-13-591Z_019e947a…jsonl`): two real review sessions burned ~$16 partly re-deriving hidden scout artifacts. Scout context preserved at `.pi/plans/2026-06-04-subagent-handoff/scout-context.md` | **Plan B** (handoff improvements) |

### Root-cause detail for #1 (the planner exit-1 cluster)

The fatal combination was: global **git-installed** copy
(`~/.pi/agent/git/github.com/Mathuv/pi-interactive-subagents`) + **project-local** extension
(`.pi/settings.json` → `../pi-extension/subagents/index.ts`) in this repo. Both registered
`subagent`, `subagent_interrupt`, `subagents_list`, `subagent_resume`; pi treats tool conflicts
between different extension paths as error diagnostics and exits 1 before the session file is
created.

Current state is safe *by accident*: the git copy was uninstalled and the global package now
points at this working tree, so both references resolve to the **same absolute file** and no
conflict is flagged (verified empirically: `pi --offline --list-models` in this repo exits 0
with no diagnostics). Any future dual-copy setup (reinstall from git, second clone, another
machine) re-triggers the crash — hence extension-side resilience.

---

## Fixes

All paths relative to the repo root. Branch off `main` (`b26732b`).

### Step 0 — this document

Persist the plan here before code changes (done).

### Fix 1 — Double-registration guard (`pi-extension/subagents/index.ts`, `subagent-done.ts`)

In `subagentsExtension(pi)`, before registering anything, check a
`Symbol.for("pi-subagents/registered")` sentinel on `globalThis` (same pattern as the existing
`/reload` interval guards at the top of `index.ts`). If another live copy already set it during
this startup, skip **all** `registerTool` / `registerCommand` / `registerMessageRenderer` calls
(first copy wins) and return.

- Clear the sentinel on `session_shutdown` so `/reload` re-registers cleanly. **Verify during
  implementation** (against `@earendil-works/pi-coding-agent/dist/core/extensions/runner.js`)
  that reload fires `session_shutdown`; if not, key the sentinel on something reload-scoped.
- Apply the same guard to `subagent-done.ts` (`subagent_done`, `caller_ping`).

→ verify: unit test invoking the factory twice with a stub `pi` (second invocation registers
zero tools/commands); E2E dup-copy repro (Verification step 3) no longer crashes the child.

### Fix 2 — Real startup stderr in failure reports (`index.ts`)

In `watchSubagent()`: when `result.exitCode !== 0 && !existsSync(sessionFile)`, call
`readScreen(surface, ~100)` **before** `closeSurface()`, strip the `__SUBAGENT_DONE_N__`
sentinel line, and use the captured output as the summary:
`Sub-agent process failed to start (exit N). Terminal output: …`.

In `resolveResultPresentation()`: only append the `Session:` / `Resume:` footer when the
session file actually exists — thread an existence flag through `SubagentResult` rather than
re-stat-ing in the formatter. When absent, state that no session file was created.

→ verify: unit test — missing-session result omits the Resume footer and says no session file
was created; E2E with the guard forced off shows the real conflict stderr in the result box.

### Fix 3 — cwd-aware agent discovery + unknown-agent error (`index.ts`)

- `loadAgentDefaults(agentName, explicitCwd?)`: when the tool call passes `cwd`, search
  `<resolved cwd>/.pi/agents/<name>.md` **first**, then parent `process.cwd()/.pi/agents`,
  then global `~/.pi/agent/agents`, then bundled `agents/`. Use `params.cwd` resolved to an
  absolute path — not `agentDefs.cwd` — to avoid the chicken-and-egg with defaults.
- In the `subagent` tool `execute()`, before `launchSubagent`: if `params.agent` is set and no
  definition was found anywhere, return an error result listing the searched paths (fail fast,
  before any surface is created).
- Update the `agent` param description and the `subagents_list` description to mention
  target-cwd lookup.

→ verify: unit tests — `loadAgentDefaults` finds a def in a tmp-dir `.pi/agents/x.md` via
explicit cwd; unknown agent name returns an error listing searched paths (no surface created).

### Fix 4 — Model preflight validation (`index.ts`)

In `execute()` before launch: validate `effectiveModel` (the `provider/id` part, without the
`:thinking` suffix) against pi's model registry, so a bad model fails the tool call instantly
instead of after a 4-second subagent crash. **Investigate first** whether
`ExtensionContext`/`ExtensionAPI` exposes the model registry (check
`dist/core/extensions/types.d.ts`). If exposed, fail with the error plus near-match
suggestions; if not exposed, skip this fix and document the limitation (do **not** shell out to
`pi --list-models` per spawn).

→ verify: spawn with `model: "nonexistent/foo"` → instant tool error, no pane created
(Verification step 5); valid-model spawn unaffected (step 4).

### Fix 5 — moved to Plan C

Auto-resume on transient provider drops is now a standalone plan:
[2026-06-11-auto-resume-transient-drops.md](./2026-06-11-auto-resume-transient-drops.md).
It executes after Plan B (both rework the `watchSubagent` result path). The fix number is
retained here to keep references stable.

### Fix 6 — Cosmetic + hardening

- **6a Cancelled presentation**: the cancel path sets `error: "cancelled"`. Thread a
  `cancelled` flag into details; `resolveResultPresentation` says
  `Sub-agent "X" cancelled after Ns.`; the `subagent_result` renderer shows status "cancelled"
  with neutral (non-error) styling.
- **6b `loadStatusConfig` hardening** (`status.ts`): when **both** config files are missing,
  return `{ enabled: false, lineLimit: DEFAULT_STATUS_LINE_LIMIT }` instead of throwing. Keep
  throwing on invalid JSON/schema.

→ verify: unit tests — cancelled result renders as `cancelled` with neutral styling and no
failure wording; `loadStatusConfig` with both files missing returns `{ enabled: false, … }`,
invalid JSON still throws.

### Maintenance — dependency drift

Bump devDependencies from `@mariozechner/pi-coding-agent ^0.65.0` to the runtime package
(`@earendil-works/pi-coding-agent` 0.79.x) so type-checks and tests run against the real API.
**Verify first** that the `@earendil-works` package exposes the same API surface and how pi's
extension loader aliases the `@mariozechner/pi-coding-agent` import specifier; if aliasing is
unclear, keep the import specifiers and bump versions only.

→ verify: type-check passes against the bumped package; `node --test test/test.ts` green;
`pi --offline --list-models` in this repo still exits 0 with no diagnostics.

---

## Files to modify

- `pi-extension/subagents/index.ts` — Fixes 1–4, cancelled flag
- `pi-extension/subagents/subagent-done.ts` — Fix 1 guard
- `pi-extension/subagents/status.ts` — Fix 6b
- `test/test.ts` — new unit tests (export new helpers via the existing `__test__` object)
- `package.json` — dependency bump

## Tests (node:test, extend `test/test.ts`)

- Registration guard: invoke the factory twice with a stub `pi`; the second invocation
  registers zero tools/commands.
- `resolveResultPresentation`: missing-session variant omits the Resume footer; cancelled
  variant renders as cancelled.
- `loadAgentDefaults` with explicit cwd (tmp-dir fixture containing `.pi/agents/x.md`).
- Unknown-agent error path.
- `loadStatusConfig` with both files missing → `{ enabled: false }`.

## Verification

1. `node --test test/test.ts` — all pass.
2. Type-check after the dependency bump.
3. **E2E repro of the original crash:** in a scratch project, add a throwaway
   `.pi/extensions/dup.ts` that re-exports the subagents extension (second copy, different
   path). From a pi session there, spawn a subagent without deny-tools. Before fixes: child
   dies, result says "exited with code 1" with a bogus Resume pointer. After fixes: no crash
   (guard); with the guard forced off, the result contains the real conflict stderr and no
   Resume footer.
4. E2E happy path: spawn a scout (`opencode-go/deepseek-v4-flash`), confirm a normal
   completion result.
5. Model preflight: spawn with `model: "nonexistent/foo"` → instant tool error, no pane
   created.
6. Integration tests if a mux is available:
   `node --test --test-concurrency=1 test/integration/*.test.ts`.

## Out of scope

- Upstream PR to HazAT (fork-only for now; keep commits clean enough to upstream later).
- `~/.pi/agent/agents.json` model overrides (already remediated).
- Stale/environmental items (#9).
- Pi-core behavior (exit-1 on conflict diagnostics) — extension-side resilience only.
- Subagent handoff improvements (#10) — Plan B, separate session.
- Auto-resume on transient drops (#6) — Plan C, separate session after Plan B.
