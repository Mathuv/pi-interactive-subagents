---
name: plan
description: >
  Planning workflow. Runs a pre-flight scout, then spawns the planner agent
  which clarifies WHAT to build and figures out HOW, with the ability to
  spawn its own scouts/researchers mid-session. Use when asked to "plan",
  "brainstorm", "I want to build X", or "let's design". Requires the
  subagents extension and a supported multiplexer (cmux/tmux/zellij).
---

# Plan

A planning workflow. A scout maps the relevant codebase, then an interactive planner clarifies intent + requirements and designs the technical approach, producing a `plan.md` and todos.

**Announce at start:** "Let me take a quick look, then I'll send a scout to map the codebase before we start the planning session."

---

## The Flow

```
Phase 1: Quick Assessment (main session — 30s orientation)
    ↓
Phase 2: Scout (autonomous — codebase context)
    ↓
Phase 3: Spawn Planner Agent (interactive — clarifies WHAT, plans HOW, creates todos)
    ↓
    (Planner may spawn its own scouts/researchers mid-session as needed)
    ↓
Phase 4: Review Plan & Todos (main session)
    ↓
Phase 5: Execute Todos (workers — receive plan + scout context)
    ↓
Phase 6: Review
```

> **Wait discipline.** When a subagent's result gates your next step (scout → planner, planner → workers), stop and wait for it — do not redo its work in the parent session. Results steer back to you automatically when the subagent finishes; spending expensive parent tokens re-deriving what a scout or planner is already producing just duplicates effort. Parallelise only genuinely independent work.

---

## Phase 1: Quick Assessment

Quick orientation — just enough to give the scout a focused mission:

```bash
ls -la
find . -type f -name "*.ts" | head -20  # or relevant extension
cat package.json 2>/dev/null | head -30
```

Spend ~30 seconds. Tech stack, project shape, and the area relevant to the user's request. This tells you what to ask the scout to focus on.

---

## Artifact Paths

For a planning run, pick a short `<name>` (e.g. `auth-redesign`) and use a shared directory under `.pi/plans/YYYY-MM-DD-<name>/` for every deliverable. Pass explicit paths in each subagent's task and read them back with the plain `read` tool when a subagent finishes.

Standard filenames:

- `.pi/plans/YYYY-MM-DD-<name>/scout-context.md`
- `.pi/plans/YYYY-MM-DD-<name>/plan.md`
- `.pi/plans/YYYY-MM-DD-<name>/review-TODO-xxxx.md` (per-todo reviewer output, when mode is `full` or `ask-me`)
- `.pi/plans/YYYY-MM-DD-<name>/review.md` (final reviewer output, when mode is `full`, `final-only`, or `ask-me`)

---

## Phase 2: Scout

**Always spawn a scout before the planner.** The scout's context feeds into the planning session — it lets the planner skip re-asking questions whose answers live in the code, and gives it a solid base to design from.

```typescript
subagent({
  name: "🔍 Scout",
  agent: "scout",
  task: `Analyze the codebase for [user's request area]. Map file structure, key modules, patterns, conventions, and existing code related to [feature area]. Focus on what a planner would need to understand before designing this feature.

Save your findings to: .pi/plans/YYYY-MM-DD-<name>/scout-context.md`,
});
```

**Wait for the scout to finish — don't start planning in the parent meanwhile.** Its result steers back automatically. Read the scout's context file with the `read` tool — you'll pass it to the planner.

The planner can spawn **additional** scouts or researchers mid-session if it hits a factual gap. That's expected — don't try to pre-scout every possible area.

---

## Phase 3: Spawn Planner Agent

Spawn the interactive planner with the scout's context and the user's request. The planner handles everything from here: clarifying intent, compact requirements engineering, ISC, approach exploration, design validation, premortem, plan artifact, and todos.

```typescript
subagent({
  name: "💬 Planner",
  agent: "planner",
  interactive: true,
  task: `Plan: [what the user wants to build]

Scout context:
[paste scout findings here — file structure, conventions, patterns, relevant code]

Save the final plan to: .pi/plans/YYYY-MM-DD-<name>/plan.md
Create todos tagged with: <name>

For every todo, prefix the title with [tdd] for behavior-changing work (features, bug fixes, logic/API changes) or [no-tdd] for non-behavioral work (config, docs, scaffolding, type-only/refactor-only changes). Always add these markers — the orchestrator selects TDD mode after planning and uses them for smart mode filtering.`,
});
```

**The user works with the planner — wait for it; don't draft the plan in parallel in the parent.** Its plan + todos steer back automatically when the session closes. It will clarify requirements lightly (1-2 rounds of questions, not a deep spec session), propose approaches, validate the design, run a premortem, write the plan, and create todos with mandatory code examples.

When done, the user presses Ctrl+D and the plan + todos are returned to the main session.

### The planner may spawn its own specialists

During the session, the planner can spawn:
- **`scout`** — when a design decision depends on existing code it hasn't read
- **`researcher`** — when a decision depends on external facts (library tradeoffs, best practices, API behaviors)

These are internal to the planning session. You'll see them in the multiplexer but don't need to intervene.

### Optional: extra scout after planning

If the planner significantly changed scope (new subsystems, areas the original scout didn't cover), spawn another scout targeting the new areas before workers start:

```typescript
subagent({
  name: "🔍 Scout (updated scope)",
  agent: "scout",
  task: "The plan changed scope. Gather context for [new areas]. Read the plan at [plan path]. Focus on [specific files/modules the planner identified that weren't in the original scout].",
});
```

Fold the new context into the worker tasks.

---

## Phase 4: Review Plan & Todos

Once the planner closes, read the plan and list todos:

```typescript
todo({ action: "list" });
```

Review with the user:

> "Here's what the planner produced: [brief summary]. Ready to execute, or anything to adjust?"

### Review Mode Selection

Before starting execution, ask the user which review mode to use:

> "What review mode for this run?"
> - **`full`** — auto-review after every todo + final review at the end
> - **`final-only`** — skip per-todo reviews, run final review after all workers
> - **`ask-me`** — I'll prompt you after each worker and at the end
> - **`none`** — no reviews at all
>
> Default: `full`

Each review is a **single combined review**: the reviewer checks spec compliance first, then code quality/security/correctness. Do not spawn separate spec-compliance and quality reviewers unless the user explicitly asks for split reviews.

### TDD Mode Selection

> "What TDD mode for this run?"
> - **`on`** — all todos get TDD enforcement: workers write a failing test before implementing
> - **`smart`** — only behavior-changing todos (marked `[tdd]` by planner) get TDD enforcement; config/docs/refactor todos skip it
> - **`off`** — no TDD enforcement, current behavior
>
> Default: `smart`

Remember both modes — they govern behavior in Phase 5 and Phase 6.

---

## Phase 5: Execute Todos

> **Review mode:** [echo the mode chosen in Phase 4 here, e.g. `full` / `ask-me` / `final-only` / `none`]
> **TDD mode:** [echo the mode chosen in Phase 4 here, e.g. `on` / `smart` / `off`]

Spawn workers sequentially. Each worker gets the plan path and scout context. After each worker, run the review gate based on the chosen mode:

```typescript
// Workers execute todos sequentially — one at a time
// Review mode: [mode] — applied after each worker below
subagent({
  name: "🔨 Worker 1/N",
  agent: "worker",
  task: "Implement TODO-xxxx. Mark the todo as done. Plan: [plan path]\n\nScout context: [paste scout summary from Phase 2, plus any re-scout from Phase 3]",
});

// After worker finishes — check for failure first:
// If the worker crashed or exited non-zero:
//   → Do NOT run the review gate
//   → Surface the error to the user
//   → Ask: "Worker failed on TODO-xxxx. Retry, skip, or abort?"
// If the worker succeeded — run review gate based on mode:
```

**Always run workers sequentially in the same git repo** — parallel workers will conflict on commits.

### TDD Enforcement

Before spawning each worker, check whether TDD mode applies to that todo:

- **`on`** → always append the TDD block to the worker task text
- **`smart`** → append the TDD block if the todo title/body contains `[tdd]`, OR if the todo has neither `[tdd]` nor `[no-tdd]` marker (unmarked = tdd-applicable)
- **`off`** → do not append anything (identical to current behavior)

When TDD mode is active for a todo, append this block to the worker task text:

```
TDD Mode: ON for this todo.
Before implementing, write a failing test that captures the expected behavior.
Run the test to confirm it fails. Then implement until the test passes.
Commit sequence: test first (failing), then implementation (passing).
If no test framework exists yet, set one up as part of this todo.
```

### Per-Todo Review Gate

After each worker finishes, apply the review gate:

- **`full`** → auto-spawn one reviewer for spec compliance + quality, scoped to `git diff HEAD~1..HEAD`, save output to `review-TODO-xxxx.md`
- **`ask-me`** → prompt user: "Run a spec + quality review for TODO-xxxx?" — spawn reviewer if yes
- **`final-only`** / **`none`** → skip

Before spawning the reviewer, read the todo body so the reviewer can check spec compliance without needing the todo tool:

```typescript
todo({ action: "get", id: "TODO-xxxx" });
```

**Reviewer task template (single reviewer checks spec compliance + quality):**

```typescript
subagent({
  name: "🔍 Review TODO-xxxx",
  agent: "reviewer",
  task: `Review TODO-xxxx.

Pass 1 — Spec compliance:
Compare the diff against the TODO body, plan, and any relevant ISC/acceptance criteria. Flag missing requirements, out-of-scope behavior, or plan drift as P1 unless the mismatch creates a P0 production/security risk.

Pass 2 — Quality/security/correctness:
Review for bugs, security issues, maintainability traps, and correctness problems introduced by this commit.

Diff scope: git diff HEAD~1..HEAD
Plan: [plan path]
TODO body:
[paste todo({ action: "get", id: "TODO-xxxx" }) body here]

Save findings to: .pi/plans/YYYY-MM-DD-<name>/review-TODO-xxxx.md`,
});
```

When TDD mode is `on`, or `smart` and this todo is tdd-applicable, append a third pass to the reviewer task:

```
Pass 3 — TDD evidence (P2 only):
Check that test files were added/modified covering the behavior introduced in this commit. Flag missing test coverage as P2 (advisory). Do NOT flag as P0/P1 — TDD non-compliance is guidance, not a blocker.
```

### P0/P1 Fixup Loop

If the per-todo review finds P0/P1 issues or spec-compliance gaps:

1. Create a fixup todo for the issues found
2. Spawn a fixup worker to address them
3. Run the review gate again for the fixup (same mode, same diff scope)
4. **Cap fixup attempts at 2 per original todo.** After 2 failed attempts, surface the remaining issues to the user and ask whether to proceed or stop.

Fixup workers follow the same review mode as the rest of the plan — no special-casing.

```typescript
// Fixup worker (spawned after P0/P1/spec-compliance issues found in per-todo review)
subagent({
  name: "🔧 Fixup Worker (TODO-xxxx attempt 1/2)",
  agent: "worker",
  task: "Fix P0/P1/spec-compliance issues found in review of TODO-xxxx. Issues: [paste P0/P1/spec gap list]. Plan: [plan path]",
});
```

When TDD mode is `on`, or `smart` and the original todo was tdd-applicable, append to the fixup worker task:

```
Include a regression test that covers the fix. If fixing a failing test, ensure it passes after your change.
```

---

## Phase 6: Final Review

> **Review mode:** [echo the mode chosen in Phase 4 here]

After all todos are complete, run the final review gate based on the chosen mode:

- **`full`** or **`final-only`** → auto-spawn one reviewer for final spec compliance + quality, save output to `review.md`
- **`ask-me`** → prompt user: "Run a final spec + quality review of all changes?" — spawn reviewer if yes, save to `review.md`
- **`none`** → skip — go directly to the completion checklist

Before spawning the final reviewer, collect todo context so the reviewer can check spec compliance without needing the todo tool:

```typescript
todo({ action: "list" });
// Paste completed todo IDs/titles and any important acceptance criteria into the reviewer task.
```

When spawning the final reviewer:

```typescript
subagent({
  name: "Reviewer (final)",
  agent: "reviewer",
  interactive: false,
  task: `Review all changes in this plan.

Pass 1 — Spec compliance:
Compare the completed implementation against the plan, todo summary, ISC, and accepted scope. Flag missing requirements, out-of-scope behavior, or plan drift as P1 unless the mismatch creates a P0 production/security risk.

Pass 2 — Quality/security/correctness:
Review for bugs, security issues, maintainability traps, integration issues, and correctness problems across the full implementation.

Plan: [plan path]
Todo summary:
[paste completed todo IDs/titles and important acceptance criteria here]

Save findings to: [plan dir]/review.md`,
});
```

When TDD mode is not `off`, append a third pass to the final reviewer task:

```
Pass 3 — TDD evidence (P2 only):
Check that behavior-changing commits have associated test files. Flag missing coverage as P2. This is advisory — do not block on it.
```

Triage findings:

- **P0** — Real bugs, security issues → fix now
- **P1** — Spec-compliance gaps, genuine traps, maintenance dangers → fix before merging
- **P2** — Minor issues → fix if quick, note otherwise
- **P3** — Nits → skip

Create todos for P0/P1, run workers to fix, re-review only if fixes were substantial.

---

## ⚠️ Completion Checklist

Before reporting done:

1. ✅ Scout ran before the planner?
2. ✅ Scout context was passed to the planner?
3. ✅ All worker todos closed?
4. ✅ Every todo has a polished commit (using the `commit` skill)?
5. ✅ Spec-compliance + quality reviews completed per chosen mode? (`full` → per-todo + final ran; `final-only` → final ran; `ask-me` → user was prompted at each gate; `none` → skipped intentionally)
6. ✅ TDD mode followed per chosen setting? (`on` → all workers got TDD instructions; `smart` → applicable todos got TDD instructions; `off` → no TDD language injected)
7. ✅ Review findings and spec gaps triaged and addressed (if any reviews ran)?
