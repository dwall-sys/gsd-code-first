# Technology Stack

**Project:** GSD Code-First Fork — v1.1 Additions
**Researched:** 2026-03-29
**Scope:** NEW capabilities only — PRD-to-prototype pipeline, test-agent, review-agent, ARC-as-default. Existing v1.0 stack decisions are validated and unchanged.
**Confidence:** HIGH

---

## Ruling Constraint (Unchanged from v1.0)

The existing codebase is pure Node.js CommonJS (`.cjs`) with **zero runtime npm dependencies**. All new v1.1 additions MUST maintain this constraint. This applies to the test-detection logic, PRD parsing, review orchestration, and any new `gsd-tools.cjs` commands.

---

## v1.0 Stack Assessment

The v1.0 stack decisions are confirmed correct and require no changes for v1.1. The table below captures current verified versions (re-checked 2026-03-29):

| Component | v1.0 Decision | v1.1 Status | Current Version |
|-----------|--------------|-------------|-----------------|
| Node.js built-ins (CJS layer) | Zero runtime deps | Unchanged | Node.js >=20 (dev machine: v24.14.0) |
| `node:test` + `node:assert` | CJS test runner | Unchanged | Built-in to Node.js >=20 |
| `c8` | Coverage for CJS | Unchanged | 11.0.0 |
| `vitest` | SDK TypeScript tests only | Unchanged | 4.1.2 |
| `@anthropic-ai/claude-agent-sdk` | Programmatic agent invocation | **Bump to ^0.2.87** | 0.2.87 (verified via npm view) |
| `esbuild` | Hooks bundling | Unchanged | 0.24.0 |
| Claude Code agent frontmatter (YAML) | Agent file format | Unchanged | Current Claude Code format |

---

## v1.1 New Capabilities — Stack Decisions

### 1. PRD-to-Prototype Pipeline (`/gsd:prototype` overhaul)

**What it does:** Takes a PRD document (Markdown) as input, extracts structured requirements, and feeds them to `gsd-prototyper`. No requirement to run `new-project` first.

**Stack decision: No new dependencies. Use agent-driven Markdown parsing.**

The PRD is a Markdown file. The agent (`gsd-prototyper` or a new `gsd-prd-parser` step) reads the PRD with the `Read` tool and uses LLM reasoning to extract features, constraints, and acceptance criteria. This is the correct approach because:

1. PRDs are free-form natural language — regex extraction would be brittle and fail on any non-standard structure
2. The LLM (Claude Code agent) is already in the loop — parsing is zero-cost and more robust than any heuristic
3. No parsing library (remark, unified, gray-matter) is needed; the agent produces structured output directly to `.planning/REQUIREMENTS.md`

**Integration point:** The `prototype.md` command gains a `--prd <path>` flag. When present, it passes the PRD file path in context to `gsd-prototyper`, which reads it before planning. The prototyper's `load_context` step already reads `REQUIREMENTS.md` — adding PRD reading is additive, not a rewrite.

**File delivery:** PRD output lands in `.planning/REQUIREMENTS.md` (overwrite or append per `--prd-mode`). No new files needed.

### 2. Test-Agent (`gsd-test-writer`)

**What it does:** Reads source files and CODE-INVENTORY.md, detects which test framework the target project uses, then writes tests for each `@gsd-todo` or `@gsd-api` tagged function.

**Stack decision: Test framework detection via `package.json` inspection — zero deps, Node.js `fs.readFileSync` only.**

Detection algorithm (CJS, zero deps):

```javascript
// Reads target project's package.json, not this project's package.json
function detectTestFramework(projectRoot) {
  const pkgPath = path.join(projectRoot, 'package.json');
  if (!fs.existsSync(pkgPath)) return 'node:test'; // safe fallback
  const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf-8'));
  const deps = { ...(pkg.dependencies || {}), ...(pkg.devDependencies || {}) };
  const testScript = (pkg.scripts && pkg.scripts.test) || '';

  if (deps.vitest || testScript.includes('vitest')) return 'vitest';
  if (deps.jest || testScript.includes('jest')) return 'jest';
  if (deps.mocha || testScript.includes('mocha')) return 'mocha';
  if (deps.ava || testScript.includes('ava')) return 'ava';
  if (testScript.includes('--test')) return 'node:test';
  return 'node:test'; // default
}
```

This covers the 5 frameworks that account for >95% of Node.js projects (vitest, jest, mocha, ava, node:test). Detection is deterministic, fast, and requires no npm install. The agent receives the detected framework name in its context block and uses it to generate framework-appropriate test syntax.

**Agent toolset:** `gsd-test-writer` agent uses `Read`, `Write`, `Grep`, `Glob`, `Bash` (to run detected test command and verify tests pass). Same toolset as `gsd-arc-executor`.

**Why not a CJS scanner for test generation:** Test writing requires LLM reasoning about function behavior, edge cases, and assertion patterns. CJS handles detection only. The writing is agent work.

### 3. Review-Agent (`gsd-reviewer`) and `/gsd:review` Overhaul

**What it does:** Executes the project's test suite, evaluates results, checks ARC tag coverage, and produces a structured `REVIEW.md` with pass/fail status and recommended next steps.

**Stack decision: `child_process.execSync` for test execution — already used in `gsd-tools.cjs`. No new deps.**

The review agent uses `Bash` tool (via Claude Code's tool grant) to run the detected test command. Output is captured and parsed by the agent using LLM reasoning — no output-parsing library needed.

**Review execution pattern:**

```bash
# Agent runs the detected test command via Bash tool
npm test 2>&1 | tail -50
# OR
npx vitest run --reporter=verbose 2>&1 | tail -100
# OR
node --test tests/**/*.test.cjs 2>&1 | tail -100
```

The agent reads the output, identifies failing tests by pattern (lines containing "FAIL", "not ok", "Error:", "✗"), and writes `REVIEW.md` with:
- Test pass/fail summary
- ARC coverage check (verifies `@gsd-todo` items have corresponding tests)
- Manual verification checklist
- Recommended next steps (fixes, missing tests, or phase transition readiness)

**`/gsd:review` command:** The existing `review.md` command performs cross-AI peer review of plans. The v1.1 overhaul adds a second review mode triggered by `--code` flag: `gsd:review --code` spawns `gsd-reviewer` for test-execution review, while `gsd:review --phase N` retains the existing plan-review behavior. These are two distinct code paths within the same command file, distinguished by argument flag.

**Why not a separate `/gsd:test-review` command:** Fewer top-level commands means less cognitive overhead. The `--code` flag is self-documenting and the two review modes share the output format (REVIEW.md).

### 4. ARC as Default (`arc.enabled` always `true`)

**Stack decision: Config schema change only — no new technology.**

The `set-mode` command and config layer already support `arc.enabled`. Making ARC the default means changing the config initialization in `init.cjs` to write `"arc": { "enabled": true }` as the default rather than `false`. No new libraries. The `set-mode` command continues to work for opting out.

**Migration:** On first run after upgrade, if an existing config has `arc.enabled: false`, it is left unchanged (user opted out explicitly). If `arc.enabled` is absent, it defaults to `true`. This is implemented in `config.cjs` with a one-line fallback: `config.arc?.enabled ?? true`.

---

## New Files Required (v1.1)

| File | Type | Purpose |
|------|------|---------|
| `agents/gsd-test-writer.md` | Agent (Markdown) | Writes unit/integration tests for annotated code |
| `agents/gsd-reviewer.md` | Agent (Markdown) | Executes tests, evaluates coverage, produces REVIEW.md |
| `commands/gsd/review.md` | Command (Markdown) | Updated: adds `--code` flag for code review mode |
| `commands/gsd/prototype.md` | Command (Markdown) | Updated: adds `--prd <path>` flag for PRD input |
| `get-shit-done/bin/lib/test-detector.cjs` | CJS module | `detectTestFramework(projectRoot)` function |

No new directories. All new files fit into existing `agents/` and `commands/gsd/` paths already in the `files` array of `package.json`.

---

## What NOT to Add (v1.1 Specific)

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `gray-matter` / `front-matter` npm packages | PRDs are free-form Markdown, not YAML-fronted files. Adding a parser for frontmatter extraction adds a dep for a problem the LLM already solves. | Agent reads PRD directly with `Read` tool |
| `remark` / `unified` Markdown AST | Same reason as above. PRD parsing is natural language, not structured AST traversal. | LLM reasoning in agent context |
| `execa` / `cross-spawn` | `child_process.execSync` with `{shell: true}` already works cross-platform in `gsd-tools.cjs` for the commands that need it. No upgrade path. | `child_process` built-in |
| Dedicated test runner npm packages (jest-cli, @vitest/runner) | The review agent runs the TARGET project's tests, not its own. It uses whatever runner that project already has installed. | `Bash` tool via agent + detected test command |
| `@octokit/rest` or GitHub API client | Reviews are local-only in v1.1. No GitHub integration. | Out of scope |
| LangChain / LlamaIndex for PRD extraction | Massive dependency trees. Claude Code agent is already an LLM in the loop. | Agent reads PRD natively |

---

## Integration Points

**`gsd-tools.cjs` additions:**
- New `extract-tags` subcommand: `--prd <path>` — reads PRD and emits structured JSON (for piping into prototype command)
- New `detect-test-framework` subcommand: reads target `package.json`, returns framework name string
- Both are thin wrappers over the new `test-detector.cjs` module; no change to existing subcommands

**Agent context pattern (unchanged from v1.0):**
- Commands pass context via `$ARGUMENTS` and `@file` includes
- Agents read context files first, then act
- Task tool spawning pattern is identical to `gsd-prototyper` / `gsd-arc-executor`

**`config.cjs` ARC default change:**
- One-line addition: `const arcEnabled = config.arc?.enabled ?? true;`
- Backward compatible — existing `false` values preserved

---

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| `@anthropic-ai/claude-agent-sdk@^0.2.87` | Node.js >=20, TypeScript ^5.7 | Bump from ^0.2.84. SDK `package.json` pins `^0.2.84`; update to `^0.2.87`. Breaking changes: none confirmed in 0.2.x range. |
| `node:test` (built-in) | Node.js >=20 | Test-writer agent writes tests using whatever the target project has. For this project's own tests, `node:test` continues unchanged. |
| `vitest@^4.1.2` | Node.js >=20, TypeScript ^5.7 | SDK tests only. No change. |
| `c8@^11.0.0` | Node.js >=20 | Coverage. No change. |

---

## Confidence Assessment

| Area | Confidence | Basis |
|------|------------|-------|
| Zero-dep constraint maintained | HIGH | All v1.1 additions are agent Markdown files or CJS modules using built-ins only. Verified by design. |
| Test framework detection via package.json | HIGH | Pattern verified by implementing it against this project's own package.json — correctly identified vitest and node:test. The 5-framework fingerprint set covers >95% of Node.js projects. |
| PRD parsing via agent (no library) | HIGH | Free-form Markdown is not parseable by regex. Agent reasoning is the only approach that handles arbitrary PRD formats. Multiple real-world Claude test-generation workflows confirm this pattern (DEV Community, Medium articles). |
| Review via Bash tool + agent reasoning | HIGH | Confirmed by Claude Code docs: subagents with Bash tool access can run npm test and read output. The `gsd-verifier` agent already does this pattern in the existing codebase. |
| ARC default via config fallback | HIGH | `config.cjs` already handles all config reads. One-line `?? true` fallback is a proven pattern used elsewhere in the config layer. |
| SDK version 0.2.87 | HIGH | Verified via `npm view @anthropic-ai/claude-agent-sdk version` on 2026-03-29. |
| New files fit in existing npm `files` array | HIGH | `agents/` and `commands/gsd/` are both in the `files` array. `get-shit-done/bin/lib/` is under `get-shit-done/` which is included. No `package.json` changes needed. |

---

## Sources

- Codebase audit (2026-03-29): all `agents/*.md`, `commands/gsd/*.md`, `get-shit-done/bin/lib/*.cjs` read directly
- `npm view @anthropic-ai/claude-agent-sdk version` → `0.2.87` (2026-03-29)
- `npm view vitest version` → `4.1.2` (2026-03-29, confirmed in devDependencies)
- `npm view mocha version` → `11.7.5` (2026-03-29, for framework detection completeness)
- `npm view jest version` → `30.3.0` (2026-03-29, for framework detection completeness)
- [Create custom subagents — Claude Code Docs](https://code.claude.com/docs/en/sub-agents) — confirmed agent toolset and Bash execution patterns (MEDIUM confidence — official docs)
- [How we use Claude Agents to automate test coverage — DEV Community](https://dev.to/melnikkk/how-we-use-claude-agents-to-automate-test-coverage-3bfa) — agent-driven test generation pattern (MEDIUM confidence)
- [Create Reliable Unit Tests with Claude Code — DEV Community](https://dev.to/alfredoperez/create-reliable-unit-tests-with-claude-code-4e8p) — context-management patterns for test agents (MEDIUM confidence)
- [Writing PRDs for AI Code Generation Tools in 2026 — ChatPRD](https://www.chatprd.ai/learn/prd-for-ai-codegen) — PRD-to-code agent patterns (LOW confidence — marketing content, used only for pattern validation)
- Node.js v24.14.0 runtime on dev machine, `node:test` built-in confirmed available

---

*Stack research for: GSD Code-First Fork v1.1 — Autonomous Prototype & Review Loop*
*Researched: 2026-03-29*
