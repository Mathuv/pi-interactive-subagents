# Plan: model-preflight fixes P1 + P2 (post-review, Plan A branch)

## Context

Adversarial review of `fix/session-error-analysis-2026-06-11` surfaced two real bugs in the
Plan A model-preflight feature (`validateModelPreflight`, commit `a5e2f0e`), both verified
against pi 0.79.1 dist sources:

- **P1** — `pi-extension/subagents/index.ts:529` strips at the FIRST colon unconditionally
  (`effectiveModel.split(":")[0]`). Pi's real resolver (`dist/core/model-resolver.js`,
  `parseModelPattern` ~152) tries the FULL pattern first, then splits on the LAST colon, and
  only strips when the suffix is a valid thinking level (`off|minimal|low|medium|high|xhigh`,
  `dist/cli/args.js:6`). Result: valid colon-id models (Bedrock `amazon.nova-lite-v1:0`,
  OpenRouter `:exacto`) are falsely rejected before launch.
- **P2** — preflight call site (`index.ts:1766-1782`) validates against the PARENT's
  `ctx.modelRegistry`, but when the target cwd has its own `.pi/agent/`, the child gets
  `PI_CODING_AGENT_DIR=<cwd>/.pi/agent` (`index.ts:1410-1416`) and resolves models against
  that repo's own `models.json`. Parent registry is the wrong oracle → false reject.

**Decided with Mat (evidence-verified by subagent):** literal `:thinking` is NEVER a valid pi
suffix — a `:thinking` pin reaches `buildFallbackModel`, ships the literal string to the
provider, and 400s after ~4-5s. The Plan A test asserting acceptance of `:thinking`
(test/test.ts:2312-2318) encoded a wrong assumption → **flip it to expect rejection**
(strict mirror).

**Reuse decision (Mat asked):** direct reuse of pi's `parseModelPattern` is not possible —
not exported from the package entry (`dist/index.js`), and the extension loader aliases only
bare package names, no subpaths (`dist/core/extensions/loader.js:29-46`). `ModelRegistry`
exposes no resolver method. So: ~12-line strict mirror of `parseModelPattern` semantics,
with a comment naming the mirrored source so future divergence is traceable.

Conventions: strict TDD (red → green, test+impl in one commit per fix), karpathy surgical
scope, caveman-commit messages, keep tree compiling (Mat's live pi loads this working tree).

## Step 0 — plan doc in repo

Per Mat's global rule, write this plan to `docs/2026-06-12-model-preflight-review-fixes.md`
(repo convention: dated docs committed with/before the fixes — fold into commit 1 or its own
`docs:` commit, matching Plan A's `72c9c58` pattern).

## Commit 1 — P1: mirror pi's last-colon strict parsing

**Red** — in `describe("model preflight")` (test/test.ts:2293):

1. Extend `stubRegistry()` with colon-id models:
   `{ provider: "amazon-bedrock", id: "amazon.nova-lite-v1:0" }`,
   `{ provider: "openrouter", id: "moonshotai/kimi-k2.5:exacto" }` (also exercises
   slash-in-id → provider split stays first-slash).
2. New `it("accepts models with colons in their ids")` — RED:
   `amazon-bedrock/amazon.nova-lite-v1:0` and `openrouter/moonshotai/kimi-k2.5:exacto` → null.
3. New `it("accepts colon-id models with a trailing thinking level")` — RED:
   `amazon-bedrock/amazon.nova-lite-v1:0:high` → null.
4. New `it("accepts all valid thinking level suffixes")` — loop
   `off|minimal|low|medium|high|xhigh` on `anthropic/claude-sonnet-4-6:<level>` → null
   (pins loop semantics; green today for single suffix, keep anyway).
5. **Replace** test at 2312-2318 with `it("rejects literal :thinking — not a valid pi
   thinking level")` — RED: error non-null, suggestion mentions the real model. Keep the
   bare `openai-codex/gpt-5.4` acceptance assertion (fold into #4).
6. New `it("rejects unknown models even with a valid thinking suffix")`:
   `openai-codex/gpt-5.3-codex:high` → error with `gpt-5.4` suggestion.

Run `node --test test/test.ts` — expect exactly the new RED set (2, 3, 5).

**Green** — rewrite `validateModelPreflight` body (index.ts:529-534), mirror of
`parseModelPattern` strict mode (comment: "mirrors dist/core/model-resolver.js
parseModelPattern, strict CLI --model semantics"):

```ts
const VALID_THINKING_LEVELS = new Set(["off", "minimal", "low", "medium", "high", "xhigh"]);

const registryHas = (candidate: string): boolean => {
  const slash = candidate.indexOf("/");      // first slash — ids may contain slashes
  if (slash === -1) return false;
  return !!registry.find(candidate.slice(0, slash), candidate.slice(slash + 1));
};

let base = effectiveModel;
for (;;) {
  if (registryHas(base)) return null;
  const lastColon = base.lastIndexOf(":");
  if (lastColon === -1) break;
  if (!VALID_THINKING_LEVELS.has(base.slice(lastColon + 1))) break;  // strict: reject
  base = base.slice(0, lastColon);
}
// existing suggestion block unchanged, fed from final `base` (same first-slash split);
// error message keeps quoting the original effectiveModel
```

→ verify: full suite green (192 − 1 replaced + 4 new). Commit test+impl:
`fix(subagents): mirror pi last-colon model parsing in preflight`.

## Commit 2 — P2: skip preflight when child uses cwd-local agent dir

**Red** — tool-level tests (pattern: test.ts:2338 + `createMockExtensionApi()`):

1. `it("skips model preflight when target cwd has its own .pi/agent dir")` — RED:
   temp dir + `mkdirSync(join(dir, ".pi", "agent"), { recursive: true })`; execute
   `{ name, task, model: "childonly/some-model", cwd: dir }` with ctx
   `{ modelRegistry: stubRegistry(), sessionManager: { getSessionFile: () => null } }`;
   assert `result.details?.error !== "unknown model"` (NOT a specific downstream error —
   mux gate is environment-dependent; the sessionManager stub guarantees clean early
   return either way).
2. `it("still preflights when target cwd has no local agent dir")` — temp dir without
   `.pi/agent` → `details.error === "unknown model"` (pins skip condition to existsSync).

**Green** — call site (index.ts:1766-1782): reuse `resolveSubagentPaths(params, agentDefs)`
(index.ts:297-313, both already in scope/imported) and gate with the SAME expression as the
env decision at index.ts:1412:

```ts
const { localAgentDir } = resolveSubagentPaths(params, agentDefs);
const usesLocalAgentDir = !!(localAgentDir && existsSync(localAgentDir));
if (agentDefs?.cli !== "claude" && modelRegistry && !usesLocalAgentDir) { /* existing body */ }
```

One-line comment: child resolves against `<cwd>/.pi/agent/models.json`, parent registry is
the wrong oracle.

→ verify: full suite green (+2). Commit:
`fix(subagents): skip model preflight when child uses cwd-local agent dir`.

## Accepted trade-offs (noted, out of scope)

- Preflight skipped entirely for local-agent-dir children → bad pins there regain the old
  slow failure. False rejects worse than slow true failures.
- `registry.find` is exact-match; pi's `tryMatchModel` also does case-insensitive partial
  matching — preflight stays stricter than pi for shorthand pins. Pre-existing, unchanged.
- Skip-condition expression duplicated at preflight site and index.ts:1412 (same call,
  ms apart) — extracting shared boolean rejected for scope.
- Preflight still checks membership, not auth (`getAvailable`) — known Plan A follow-up.

## Files touched

- `pi-extension/subagents/index.ts` (preflight body ~529; call site ~1766)
- `test/test.ts` (model-preflight describe block ~2293)
- `docs/2026-06-12-model-preflight-review-fixes.md` (new)

## Verification

- Per fix: red run (`node --test test/test.ts`, only intended failures) → green run (full pass).
- Final: `git log --oneline` = docs(+optional) + 2 fix commits, each self-contained;
  diff touches only the three files above.
- Optional live probe (post-merge): in a pi session, `subagent` with
  `model: "amazon-bedrock/amazon.nova-lite-v1:0"`-style colon id no longer errors at
  preflight; `model: "anthropic/claude-sonnet-4-6:thinking"` fails instantly with suggestion.
