# Architecture Patterns

**Domain:** Node.js CLI framework with AI agent orchestration, code annotation parsing, and extensible command system
**Researched:** 2026-03-29 (v1.1 milestone update)
**Confidence:** HIGH — based on direct inspection of all existing agents, commands, workflows, and gsd-tools.cjs

---

## Existing System Architecture (v1.0 baseline — unchanged)

The upstream architecture and v1.0 Code-First extensions are documented below as a stable reference. All v1.1 additions are strictly additive to this foundation.

### Current Layer Map

```
┌─────────────────────────────────────────────────────────────┐
│  Entry Points (Claude MCP / npx invocation)                 │
│  commands/gsd/*.md  (slash-command descriptors)             │
└──────────────────────────┬──────────────────────────────────┘
                           │ @-reference injection
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Workflow Layer                                              │
│  get-shit-done/workflows/*.md  (step-by-step procedures)    │
│  - Reads STATE.md and config via gsd-tools init             │
│  - Groups plans into waves, spawns subagents                │
└────────┬─────────────────────────────┬───────────────────────┘
         │ Task() spawning             │ gsd-tools.cjs Bash calls
         ▼                             ▼
┌────────────────────┐   ┌────────────────────────────────────┐
│  Agent Layer       │   │  CLI Tools Layer                   │
│  agents/*.md       │   │  get-shit-done/bin/gsd-tools.cjs   │
│  - gsd-executor    │   │  ├── lib/core.cjs                  │
│  - gsd-planner     │   │  ├── lib/state.cjs                 │
│  - gsd-verifier    │   │  ├── lib/phase.cjs                 │
│  - gsd-arc-executor│   │  ├── lib/config.cjs                │
│  - gsd-arc-planner │   │  ├── lib/init.cjs                  │
│  - gsd-prototyper  │   │  ├── lib/roadmap.cjs               │
│  - gsd-code-planner│   │  ├── lib/template.cjs              │
│  - gsd-annotator   │   │  ├── lib/verify.cjs                │
│  - ... (23 total)  │   │  └── lib/model-profiles.cjs        │
└────────────────────┘   └──────────────────────────┬─────────┘
                                                     │
                                                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Planning State Layer                                        │
│  .planning/                                                  │
│  ├── config.json        (project config + feature flags)    │
│  ├── STATE.md           (current position, decisions)       │
│  ├── ROADMAP.md         (phase definitions)                 │
│  ├── REQUIREMENTS.md    (tracked requirements)              │
│  ├── prototype/         (code-first artifacts)              │
│  │   ├── CODE-INVENTORY.md  (extracted @gsd-tags)           │
│  │   └── PROTOTYPE-LOG.md   (build log)                     │
│  └── phases/phase-N-*/  (per-phase artifacts)               │
│      ├── PLAN.md                                            │
│      ├── SUMMARY.md                                         │
│      └── CONTEXT.md                                         │
└─────────────────────────────────────────────────────────────┘
```

### Key Architectural Properties (stable, do not break)

**Command descriptor pattern.** Every slash-command is a `.md` file with YAML frontmatter. No logic in command files — logic lives in workflow files or gsd-tools.cjs.

**Orchestrator-subagent pattern.** Workflow files call `gsd-tools.cjs init <workflow>` to load context as JSON, then spawn typed subagents via `Task()`. Each subagent has a fresh context window.

**gsd-tools.cjs as state gateway.** All reads/writes to `.planning/` go through `gsd-tools.cjs`. Agents never directly manipulate planning files.

**Wrapper pattern for upstream compatibility.** New agents wrap upstream behavior (gsd-arc-executor wraps gsd-executor) rather than modifying upstream files. Upstream files are never touched.

**Additive installer.** `bin/install.js` copies new files into `~/.claude/`. New features add files; nothing is modified in place.

---

## v1.1 Extension Architecture: PRD-to-Prototype Pipeline, Test-Agent, Review-Agent, ARC-as-Default

The four new features require three new agents, one new command, modifications to two existing commands, and one config change. All are additive.

### New Component Map (v1.1 additions only)

```
┌─────────────────────────────────────────────────────────────┐
│  v1.1 Entry Points (NEW commands)                           │
│                                                              │
│  /gsd:prototype  (MODIFIED — gains PRD ingestion step)      │
│  /gsd:review     (MODIFIED — gains test+eval behavior)      │
└──────────────┬──────────────────────────┬───────────────────┘
               │                          │
               ▼                          ▼
┌──────────────────────────┐  ┌──────────────────────────────┐
│  PRD-to-Prototype Path   │  │  Review Path                 │
│                          │  │                              │
│  1. Read PRD input       │  │  1. Run tests (gsd-tester)   │
│  2. Spawn gsd-prototyper │  │  2. Evaluate results         │
│     (ARC annotations     │  │  3. Spawn gsd-reviewer       │
│      from PRD context)   │  │  4. Produce REVIEW.md        │
│  3. Auto extract-plan    │  │  5. Surface next steps       │
└──────────────────────────┘  └──────────────────────────────┘
               │                          │
               ▼                          ▼
┌─────────────────────────────────────────────────────────────┐
│  New Agent Layer (v1.1)                                      │
│                                                              │
│  gsd-tester   — writes unit/integration tests for a phase   │
│  gsd-reviewer — evaluates test results + manual checks,     │
│                 produces REVIEW.md with next-steps           │
└─────────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│  Config Change (v1.1)                                        │
│  arc.enabled defaults to true (was opt-in)                  │
│  Affects: gsd-arc-executor and gsd-arc-planner routing      │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Boundaries

### New vs Modified: Explicit Inventory

| Component | New or Modified | File Path | What Changes |
|-----------|----------------|-----------|--------------|
| `gsd-tester` agent | NEW | `agents/gsd-tester.md` | New agent for test generation |
| `gsd-reviewer` agent | NEW | `agents/gsd-reviewer.md` | New agent for review + evaluation |
| `/gsd:review` command | MODIFIED | `commands/gsd/review.md` | Current: cross-AI plan review. v1.1: test execution + evaluation + review output |
| `/gsd:prototype` command | MODIFIED | `commands/gsd/prototype.md` | Gains PRD ingestion step before spawning gsd-prototyper |
| `arc.enabled` config default | MODIFIED | `lib/config.cjs` (schema default) | Change default from `false` to `true` |
| `bin/install.js` | MODIFIED | `bin/install.js` | Register two new agent files for copy |
| `gsd-arc-executor` agent | UNCHANGED | `agents/gsd-arc-executor.md` | Already handles ARC obligations; no change needed |
| `gsd-arc-planner` agent | UNCHANGED | `agents/gsd-arc-planner.md` | Already reads CODE-INVENTORY.md; no change needed |
| `gsd-prototyper` agent | UNCHANGED | `agents/gsd-prototyper.md` | PRD ingestion is in the command orchestrator, not the agent |
| All upstream GSD commands | UNCHANGED | `get-shit-done/commands/` | Non-negotiable constraint |

### Component Responsibilities

| Component | Responsibility | Reads From | Writes To |
|-----------|---------------|------------|-----------|
| `/gsd:prototype` (v1.1) | Ingest PRD/context, pass to gsd-prototyper, auto-run extract-plan | `$ARGUMENTS`, PRD file, PROJECT.md, REQUIREMENTS.md, ROADMAP.md | delegates to gsd-prototyper |
| `gsd-tester` agent | Write unit and integration tests for prototype code; follow RED-GREEN conventions | Prototype source files, CODE-INVENTORY.md, PROTOTYPE-LOG.md | Test files in project; test commit |
| `gsd-reviewer` agent | Execute tests, evaluate pass/fail, run manual checks, produce evaluation | Test results (stdout), prototype source files, PROTOTYPE-LOG.md | `.planning/prototype/REVIEW.md` |
| `/gsd:review` (v1.1) | Orchestrate: spawn gsd-tester → run tests → spawn gsd-reviewer → present next steps | Prototype artifacts | delegates to agents |
| `arc.enabled` default=true | ARC mode is on by default; no config required to get ARC behavior | `.planning/config.json` | Routing in gsd-arc-executor / gsd-arc-planner |

---

## Data Flow

### PRD-to-Prototype Pipeline (new)

```
User provides PRD (file path or inline text in $ARGUMENTS)
        │
        ▼
/gsd:prototype --prd <path>
  → Read PRD file, extract: goals, requirements, constraints, tech choices
  → Merge PRD context into prototype invocation prompt
  → Spawn gsd-prototyper with PRD-enriched context
        │
        ▼
gsd-prototyper builds annotated scaffold
  → Reads PROJECT.md, REQUIREMENTS.md, ROADMAP.md
  → PRD context added as additional input (phase scope, architectural decisions)
  → Writes prototype files with @gsd-tags embedded
  → Writes .planning/prototype/PROTOTYPE-LOG.md
        │
        ▼
/gsd:prototype auto-runs extract-plan
  → gsd-tools extract-tags --format md --output .planning/prototype/CODE-INVENTORY.md
        │
        ▼
.planning/prototype/CODE-INVENTORY.md ready for /gsd:iterate
```

### Test-Agent Data Flow (new)

```
/gsd:review
        │
        ▼
Step 1: Spawn gsd-tester
  → Reads prototype source files (from PROTOTYPE-LOG.md file list)
  → Reads CODE-INVENTORY.md for @gsd-api and @gsd-constraint context
  → Writes test files (unit/integration) adjacent to prototype files
  → Commits: test(prototype): add tests from gsd-tester
        │
        ▼
Step 2: Run tests
  → /gsd:review orchestrator runs test command (detected from project type)
  → Captures stdout/stderr, exit code, pass/fail counts
        │
        ▼
Step 3: Spawn gsd-reviewer
  → Receives: test results, prototype source, CODE-INVENTORY.md, PROTOTYPE-LOG.md
  → Evaluates: test coverage vs @gsd-api contracts, gaps vs @gsd-todo list
  → Runs manual verification checks (file existence, import resolution, structure)
  → Writes .planning/prototype/REVIEW.md
        │
        ▼
Step 4: Present next steps to user
```

### ARC-as-Default Flow Change (config change only)

```
Before v1.1:
  arc.enabled not set → defaults to false → gsd-executor spawned

After v1.1:
  arc.enabled not set → defaults to true → gsd-arc-executor spawned
  arc.enabled: false → explicit opt-out → gsd-executor spawned

No change to routing logic in iterate.md or execute-phase workflow.
Change is purely in lib/config.cjs default value and schema documentation.
```

### Review Artifact Flow

```
.planning/prototype/REVIEW.md structure:
  ├── Test Results section (pass/fail counts, failing test names)
  ├── Coverage Analysis (which @gsd-api contracts have test coverage)
  ├── Gap Analysis (@gsd-todo items without tests, unverified @gsd-risk items)
  ├── Manual Verification (file structure, imports resolve, stubs flagged)
  └── Next Steps (prioritized: fix failing tests / fill coverage gaps / run /gsd:iterate)
```

---

## Suggested Build Order

Dependencies flow upward. Build bottom-up to avoid blocked work.

### Layer 1: Config Change (no dependencies on new code)

**1a. `arc.enabled` default to `true` in `lib/config.cjs`**
Single-field default change. Zero risk. No new files. Enables ARC behavior for all new users without requiring a `set-mode` call. Existing projects with explicit `arc.enabled: false` are unaffected — the config override persists. Build first so all downstream testing reflects the intended default.

### Layer 2: PRD-to-Prototype Pipeline (depends on Layer 1)

**2a. Modify `/gsd:prototype` command to accept `--prd <path>` flag**
Add a PRD ingestion step before spawning gsd-prototyper. The command reads the PRD file and injects its content as additional context in the Task() call to gsd-prototyper. The gsd-prototyper agent itself does not change — the enrichment happens in the command orchestrator. This is the minimal viable PRD pipeline.

Key addition to `commands/gsd/prototype.md`:
- Parse `--prd <path>` from `$ARGUMENTS`
- If present: read PRD file, extract goals/requirements/constraints, inject into prototyper Task() prompt
- If absent: existing behavior unchanged (backward compatible)

### Layer 3: gsd-tester Agent (no dependencies on Layers 1-2)

**3a. Create `agents/gsd-tester.md`**
New agent that writes unit and integration tests for prototype code. Follows the same agent file format as all 22 existing agents (YAML frontmatter + Markdown prose).

Input sources:
- Prototype source files (paths from PROTOTYPE-LOG.md)
- CODE-INVENTORY.md (for @gsd-api contracts as test specifications)
- @gsd-constraint tags (test boundary conditions)
- PROTOTYPE-LOG.md (list of files built, requirement IDs covered)

Behavior pattern: mirrors `gsd-arc-executor`'s TDD execution section — detect test framework, write failing tests, verify RED state, implement minimal pass, verify GREEN. Since prototype code has stub implementations, tests are written against the @gsd-api contracts (not against stub return values).

Output: test files committed as `test(prototype): add tests from gsd-tester`

NOTE: This is a new agent type, not a wrapper around an existing agent. It does NOT subclass gsd-verifier (verifier checks phase goals) or the upstream add-tests workflow (that targets completed phase implementations). gsd-tester targets prototype code using ARC annotations as specifications.

### Layer 4: gsd-reviewer Agent (depends on Layer 3)

**4a. Create `agents/gsd-reviewer.md`**
New agent that evaluates test results and prototype quality, then produces REVIEW.md. Depends on gsd-tester having already run (tests must exist and have been executed before reviewer spawns).

Input sources:
- Test execution results (pass/fail counts, failing test names — provided in Task() prompt by the review command orchestrator)
- Prototype source files
- CODE-INVENTORY.md (for gap analysis against @gsd-todo and @gsd-api)
- PROTOTYPE-LOG.md (for decisions made during prototyping)

Output: `.planning/prototype/REVIEW.md` with structured sections (test results, coverage, gaps, manual checks, next steps).

NOTE: gsd-reviewer is NOT a replacement for gsd-verifier. gsd-verifier performs goal-backward phase verification after execution. gsd-reviewer evaluates prototype quality before iteration. The two agents serve different lifecycle positions and should remain separate.

### Layer 5: Modified `/gsd:review` Command (depends on Layers 3-4)

**5a. Modify `commands/gsd/review.md`**
The existing `/gsd:review` command performs cross-AI plan review (invokes Gemini, Claude, Codex CLIs). v1.1 needs a fundamentally different behavior: test execution + evaluation + structured review output.

Decision: The v1.1 review behavior is different enough from the existing cross-AI review behavior that the command needs restructuring. Two options:

- **Option A (recommended):** Make `/gsd:review` context-aware. If called with `--phase N`, it does the existing cross-AI plan review. If called without a phase (or with `--prototype`), it does the v1.1 prototype review pipeline. This preserves the existing behavior as a code path rather than replacing it.

- **Option B:** Create a new `/gsd:review-prototype` command and leave `/gsd:review` unchanged. Cleaner separation but adds a command users must discover.

Recommendation: Option A. The existing cross-AI review behavior is valuable and used. A context-aware command with backward-compatible defaults is consistent with the wrapper pattern used throughout this codebase.

The orchestration in the modified command:
1. If prototype mode: spawn gsd-tester → run tests → capture output → spawn gsd-reviewer with results → read REVIEW.md → present summary + next steps
2. If plan review mode: existing cross-AI review behavior unchanged

### Layer 6: Installer (depends on all above)

**6a. Modify `bin/install.js`**
Register `agents/gsd-tester.md` and `agents/gsd-reviewer.md` for copy into `~/.claude/agents/`. Follow the existing pattern — no logic changes needed, just two additional file registrations.

---

## Build Order Summary Table

| Layer | Component | Type | Depends On | Risk |
|-------|-----------|------|-----------|------|
| 1a | arc.enabled default=true | Config change | — | Low |
| 2a | /gsd:prototype --prd flag | Command modify | — | Low |
| 3a | gsd-tester agent | New agent | — | Medium |
| 4a | gsd-reviewer agent | New agent | 3a (conceptually) | Medium |
| 5a | /gsd:review command overhaul | Command modify | 3a, 4a | Medium |
| 6a | bin/install.js additions | Installer modify | 3a, 4a | Low |

Layers 2a and 3a have no inter-dependencies and can be built in parallel. Layer 4a can begin immediately after 3a's agent file shape is settled (the reviewer references the tester's output format). Layer 5a is the integration point — build last among code changes.

---

## Patterns to Follow

### Pattern: Command as thin descriptor (unchanged)
Commands contain no logic. New `/gsd:review` orchestration goes in a workflow file (`get-shit-done/workflows/review-prototype.md` or as a new section within the command using inline steps). The command file references the workflow via `@`-reference.

Exception: `/gsd:iterate` and `/gsd:prototype` embed their orchestration directly in the command file (no separate workflow file). The v1.1 changes to these commands should follow the same inline-steps pattern already established, not introduce a new workflow file indirection.

### Pattern: gsd-tools.cjs as state gateway (unchanged)
All writes to `.planning/prototype/REVIEW.md` should go through a gsd-tools command. If no suitable command exists, add `write-review` or `scaffold review` subcommands to gsd-tools.cjs rather than having agents write files directly. (Note: PROTOTYPE-LOG.md is currently written directly by gsd-prototyper via the Write tool — this is an existing exception to the gateway pattern that the tester/reviewer agents should follow for their own output files.)

### Pattern: Wrapper agents for upstream compatibility (unchanged)
gsd-tester and gsd-reviewer are new agent types, not wrappers. They do not modify gsd-executor or gsd-verifier. If behavior from gsd-verifier is needed in gsd-reviewer, copy the relevant sections rather than calling or wrapping gsd-verifier.

### Pattern: Config-gated behavior (unchanged)
ARC default change: the `arc.enabled` config still exists and can be explicitly set to `false` to opt out. The change is only to the default value in the schema. Existing projects with explicit config are unaffected.

### Pattern: Agent frontmatter (unchanged)
New agents use identical YAML frontmatter format: `name`, `description`, `tools`, `permissionMode`, `color`. Permitted tools for gsd-tester: `Read, Write, Edit, Bash, Grep, Glob`. Permitted tools for gsd-reviewer: `Read, Write, Bash, Grep, Glob`.

---

## Anti-Patterns to Avoid

### Anti-Pattern: Replacing /gsd:review entirely
The existing cross-AI plan review behavior has active use. Replacing the command wholesale breaks users who depend on `--phase N` review behavior.

**Instead:** Make the command context-aware. Prototype review is the default (no-arg) behavior; plan review is preserved with `--phase N`.

### Anti-Pattern: gsd-tester writing tests against stub implementations
Prototype code has stub implementations (hardcoded returns, NotImplementedError). Writing tests that assert stub return values creates tests that pass immediately and become valueless after real implementation.

**Instead:** gsd-tester reads @gsd-api tags as specifications and writes tests against the contract, not against stub behavior. Tests should fail on the stub (RED) and pass only when the real implementation satisfies the contract.

### Anti-Pattern: gsd-reviewer as a gsd-verifier wrapper
gsd-verifier runs after phase execution and checks goal achievement. gsd-reviewer runs after prototype building and evaluates prototype quality before execution. They serve different phases of the workflow.

**Instead:** Keep them as independent agents. gsd-reviewer can borrow evaluation patterns from gsd-verifier's "goal-backward analysis" approach, but should not call or extend gsd-verifier.

### Anti-Pattern: PRD ingestion in gsd-prototyper agent
Putting PRD reading logic inside gsd-prototyper couples the agent to a specific input format and makes it harder to invoke the agent directly without a PRD.

**Instead:** PRD ingestion belongs in the `/gsd:prototype` command orchestrator. The command reads the PRD and enriches the Task() prompt sent to gsd-prototyper. The agent only sees "additional context" — it doesn't need to know it came from a PRD vs inline arguments.

### Anti-Pattern: Test execution inside gsd-tester
Agents should not block on long-running test suite executions. Running tests inside the tester agent means the agent context window stays open during test execution.

**Instead:** gsd-tester writes the tests and commits them. The `/gsd:review` command orchestrator runs the test command via Bash, captures the output, and passes the results to gsd-reviewer as input. Test execution belongs in the command layer, not inside an agent.

---

## Integration Points

### Internal Boundaries

| Boundary | Communication Pattern | Notes |
|----------|-----------------------|-------|
| `/gsd:prototype` → `gsd-prototyper` | Task() with enriched prompt (PRD context injected by command) | gsd-prototyper unchanged; all new info arrives as additional prompt context |
| `/gsd:review` → `gsd-tester` | Task() with prototype file list and CODE-INVENTORY.md path | Tester writes tests independently; command waits for completion |
| `/gsd:review` → test runner | Bash command (detected from project type: `npm test`, `node --test`, etc.) | Command captures stdout/stderr and exit code |
| `/gsd:review` → `gsd-reviewer` | Task() with test results injected as context | Results passed in prompt, not via file — keeps reviewer self-contained |
| `arc.enabled` default | `lib/config.cjs` schema default → `gsd-tools config-get arc.enabled` → routing in iterate.md | No new integration point; existing routing logic handles `true` value |
| `gsd-reviewer` → REVIEW.md | Agent writes directly via Write tool (same pattern as gsd-prototyper → PROTOTYPE-LOG.md) | Consistent with existing prototype artifact writing pattern |

### External Services

None. v1.1 adds no external service integrations. The existing `/gsd:review` cross-AI CLI invocation pattern (Gemini, Claude, Codex CLIs) is preserved as a code path but not modified.

---

## Scalability Considerations

This is a local CLI tool. "Scale" means large codebases and long test suites, not distributed throughput.

| Concern | Small prototype (<10 files) | Medium (10-50 files) | Large (50+ files) |
|---------|---------------------------|----------------------|-------------------|
| Test generation scope | All files in one gsd-tester pass | All files, one agent pass | Filter by PROTOTYPE-LOG.md file list; don't scan whole repo |
| Test execution time | Fast, inline | Moderate | Long-running; consider `--timeout` flag |
| gsd-reviewer context budget | All test results fit | Most fit | Truncate test output; pass summary counts + failing test names only |
| REVIEW.md size | Small | Medium | Paginate by section if needed; keep under 10KB |

For v1.1: gsd-tester scopes to the files listed in PROTOTYPE-LOG.md, not the whole codebase. This bounds test generation cost regardless of codebase size.

---

## Sources

- Upstream GSD codebase: `agents/*.md` (23 agents inspected), `commands/gsd/*.md` (63 commands inspected), `get-shit-done/workflows/*.md` (HIGH confidence — directly read)
- `get-shit-done/bin/gsd-tools.cjs` command registry (lines 1-130 inspected, HIGH confidence)
- `agents/gsd-arc-executor.md` — ARC mode gating pattern (`arc.enabled` check, config-get call) (HIGH confidence)
- `agents/gsd-prototyper.md` — PROTOTYPE-LOG.md write pattern, file creation rules (HIGH confidence)
- `commands/gsd/iterate.md` — executor routing based on arc.enabled (HIGH confidence)
- `commands/gsd/review.md` + `get-shit-done/workflows/review.md` — existing cross-AI review behavior to preserve (HIGH confidence)
- `get-shit-done/workflows/add-tests.md` — upstream test generation patterns; gsd-tester follows same RED-GREEN conventions (HIGH confidence)
- `.planning/PROJECT.md` — v1.1 target features and constraints (HIGH confidence)
- `.planning/research/FEATURES.md` v1.0 feature dependency graph (HIGH confidence)

---

*Architecture research for: GSD Code-First fork — v1.1 Autonomous Prototype & Review Loop*
*Researched: 2026-03-29*
*Supersedes: v1.0 ARCHITECTURE.md (2026-03-28)*
