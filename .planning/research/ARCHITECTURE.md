# Architecture Research

**Domain:** CLI agent framework — brainstorm command, feature map auto-aggregation, architecture-mode prototype
**Researched:** 2026-03-29
**Confidence:** HIGH (based on direct codebase inspection — all existing commands, agents, lib modules, and install.js read)

---

## Standard Architecture

The existing system has five clearly separated layers. All v1.2 features integrate into this exact stack — no new layers are introduced.

```
+---------------------------------------------------------------------+
|  ENTRY POINT LAYER — User-facing slash commands                     |
|  commands/gsd/*.md (YAML frontmatter + process steps)              |
|  +----------------+  +----------------+  +-----------------------+  |
|  | brainstorm     |  | prototype      |  | extract-plan          |  |
|  | (NEW)          |  | (MODIFIED)     |  | (POSSIBLY MODIFIED)   |  |
|  +-------+--------+  +-------+--------+  +-----------+-----------+  |
|          |                   |                       |               |
+----------+-------------------+-----------------------+---------------+
|  AGENT LAYER — Spawned via Task() from commands                     |
|  agents/gsd-*.md (YAML frontmatter + system prompt)                |
|  +----------------+  +----------------+  +-----------------------+  |
|  | gsd-           |  | gsd-feature-   |  | gsd-prototyper        |  |
|  | brainstormer   |  | mapper         |  | (MODIFIED via Task()  |  |
|  | (NEW)          |  | (NEW)          |  |  context only)        |  |
|  +----------------+  +----------------+  +-----------------------+  |
+---------------------------------------------------------------------+
|  CLI TOOLS LAYER — gsd-tools.cjs subcommand dispatch               |
|  get-shit-done/bin/gsd-tools.cjs                                   |
|  +----------------+  +----------------+  +-----------------------+  |
|  | extract-tags   |  | config-get     |  | aggregate-features    |  |
|  | (existing)     |  | (existing)     |  | (NEW case)            |  |
|  +-------+--------+  +----------------+  +-----------+-----------+  |
|          |                                           |               |
+----------+-------------------------------------------+---------------+
|  LIB LAYER — Utility modules required by gsd-tools.cjs             |
|  get-shit-done/bin/lib/*.cjs                                       |
|  +----------------+  +----------------+  +-----------------------+  |
|  | arc-scanner    |  | config         |  | feature-aggregator    |  |
|  | (existing)     |  | (existing)     |  | (NEW)                 |  |
|  +-------+--------+  +----------------+  +-----------+-----------+  |
|          |                                           |               |
+----------+-------------------------------------------+---------------+
|  PLANNING STATE LAYER — Artifacts written and read by agents       |
|  .planning/                                                        |
|  +----------------+  +-------------------+  +--------------------+  |
|  | PRD.md         |  | CODE-INVENTORY.md |  | FEATURES.md (NEW) |  |
|  | (existing)     |  | (existing)        |  |                   |  |
|  +----------------+  +-------------------+  +--------------------+  |
+---------------------------------------------------------------------+
```

### Component Responsibilities

| Component | Responsibility | Layer |
|-----------|---------------|-------|
| `commands/gsd/brainstorm.md` | Orchestrate conversation to PRD pipeline; parse flags; spawn gsd-brainstormer; display draft to user; write `.planning/PRD.md` after approval; auto-run aggregate-features | Entry point (new) |
| `agents/gsd-brainstormer.md` | Drive structured conversation to extract product intent; produce well-formed PRD with AC list; read PROJECT.md for context; return PRD as Task() response string | Agent (new) |
| `commands/gsd/prototype.md` | Existing orchestrator; receives `--arch` flag to activate architecture mode; enriches gsd-prototyper Task() prompt with arch intent | Entry point (modified) |
| `agents/gsd-prototyper.md` | Existing scaffold builder; receives arch-mode signal in Task() context; switches build strategy to skeleton-first when signaled | Agent (receives modified context, no file change required) |
| `commands/gsd/extract-plan.md` | Existing tag scanner command; optionally triggers feature-map update after inventory write | Entry point (possibly modified) |
| `agents/gsd-feature-mapper.md` | Read PRD.md, CODE-INVENTORY.md, and existing FEATURES.md; produce merged FEATURES.md reflecting current state via semantic synthesis | Agent (new) |
| `get-shit-done/bin/lib/feature-aggregator.cjs` | Parse PRDs and CODE-INVENTORY.md to extract feature signals; structural merge with existing FEATURES.md; format output | Lib module (new) |
| `gsd-tools.cjs` case `aggregate-features` | CLI entry point for feature aggregation dispatch; called by commands via Bash | CLI dispatch (new case) |
| `.planning/PRD.md` | Product Requirements Document; written by brainstorm command after user approval; read by prototype, iterate, review-code, feature-mapper | Artifact (existing, now has a write path) |
| `.planning/FEATURES.md` | Feature Map; written by gsd-feature-mapper or feature-aggregator; read by gsd-brainstormer for dedup and gsd-prototyper for context | Artifact (new) |

---

## Recommended Project Structure

New files required for v1.2, placed in the existing directory convention:

```
commands/gsd/
    brainstorm.md              (NEW) entry point for /gsd:brainstorm
    prototype.md               (MODIFIED) add --arch flag handling
    extract-plan.md            (POSSIBLY MODIFIED) optional feature-map trigger

agents/
    gsd-brainstormer.md        (NEW) conversation to PRD agent
    gsd-feature-mapper.md      (NEW) PRD + tags to FEATURES.md agent
    gsd-prototyper.md          (NOT MODIFIED — arch mode via Task() context only)

get-shit-done/bin/lib/
    feature-aggregator.cjs     (NEW) PRD and tag parsing for feature map

get-shit-done/bin/
    gsd-tools.cjs              (MODIFIED) add aggregate-features case

.planning/
    PRD.md                     (EXISTING artifact — now has a write path via brainstorm)
    FEATURES.md                (NEW artifact — produced and updated by feature pipeline)
```

### Structure Rationale

- **`commands/gsd/brainstorm.md` as a new command file:** Consistent with all 63 existing commands. YAML frontmatter declares name, description, argument-hint, and allowed-tools. The `<process>` block contains numbered steps. No new conventions needed.
- **`agents/gsd-brainstormer.md` and `gsd-feature-mapper.md` as new agent files:** Consistent with all 25 existing agents. The installer copies all `gsd-*.md` files from `agents/` automatically via wildcard scan — no explicit registration required in install.js.
- **`feature-aggregator.cjs` in `lib/`:** Consistent with `arc-scanner.cjs` and `test-detector.cjs`. A focused utility module that gsd-tools.cjs requires directly. Zero external dependencies.
- **`aggregate-features` case in gsd-tools.cjs:** Consistent with `extract-tags`, `set-mode`, `detect-test-framework`. A new switch case in the main dispatch block. Commands call it via `node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" aggregate-features`.
- **`.planning/FEATURES.md` as a new artifact:** Parallel to `CODE-INVENTORY.md`. Lives at `.planning/FEATURES.md` (not `.planning/prototype/FEATURES.md`) because it is project-level, not prototype-specific. Consumed by brainstormer for deduplication; consumed by prototyper for context.

---

## Architectural Patterns

### Pattern 1: Command-Orchestrator / Agent-Worker Split

**What:** The command file handles all orchestration — flag parsing, file I/O, tool calls, approval gates, and Bash invocations. The agent file receives a Task() prompt and focuses solely on its domain task (generate PRD, map features). Agents never parse flags or invoke gsd-tools.cjs directly.

**When to use:** Every feature in this system follows this pattern. The brainstorm command orchestrates; gsd-brainstormer generates. The command writes PRD.md; the agent returns a string.

**Trade-offs:** More moving parts, but each piece is independently testable and replaceable. Agents can be reused across commands. Context window budget is controlled at the command layer.

**Concrete application for v1.2:**
- `/gsd:brainstorm` parses `--force`, `--prd-path`, `--non-interactive` flags; reads PROJECT.md; spawns gsd-brainstormer with context; receives PRD draft string; presents to user; writes `.planning/PRD.md` after approval
- `gsd-brainstormer` receives context from Task() prompt; drives the conversation; returns a structured PRD string — never touches flags or writes files directly

### Pattern 2: Context Enrichment in Task() Prompt

**What:** The command layer prepares context (project goals, existing features, current PRDs) and injects it into the Task() prompt sent to agents, rather than having agents read everything from disk themselves. Agents receive pre-digested context.

**When to use:** When an agent needs input from multiple sources that the command can cheaply concatenate. Established by prototype.md (AC list injected into gsd-prototyper Task()) and review-code.md (test results + AC list injected into gsd-reviewer Task()).

**Trade-offs:** Command grows slightly larger. Agent is simpler and faster — fewer Read tool calls needed inside the agent.

**Concrete application for v1.2:**
- `/gsd:brainstorm` reads `.planning/FEATURES.md` (if it exists) and injects it into gsd-brainstormer Task() prompt: "Existing features: [FEATURES.md content]. Do not duplicate these — focus on gaps and new capabilities."
- `/gsd:prototype --arch` injects arch intent into gsd-prototyper Task() prompt: "Architecture mode: skeleton-first. Build minimal interface stubs with @gsd-todo tags for each component. Do not implement business logic."

### Pattern 3: Auto-Chain — Command Triggers Downstream CLI Tool

**What:** After a command completes its primary task, it runs a downstream Bash command automatically without spawning another agent. Established by `annotate.md` (auto-runs extract-plan after gsd-annotator finishes) and `prototype.md` (auto-runs extract-tags after gsd-prototyper finishes).

**When to use:** When the downstream step is deterministic, fast, and always beneficial after the primary step. Not for operations requiring LLM judgment or user input.

**Concrete application for v1.2:**
- `/gsd:brainstorm` auto-runs `aggregate-features` after writing PRD.md:
  `node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" aggregate-features --input .planning/PRD.md --output .planning/FEATURES.md`
- `/gsd:prototype` Step 5b (after existing extract-tags) optionally auto-runs `aggregate-features` to merge new @gsd-tags into FEATURES.md

### Pattern 4: gsd-tools.cjs as the Thin CLI Dispatch Shell

**What:** gsd-tools.cjs is a pure dispatch switch: parse args, require the lib module, call the exported function. All logic lives in `lib/*.cjs` modules. The switch case itself contains no business logic.

**Concrete application for v1.2:**

The `aggregate-features` case follows exactly the same pattern as `extract-tags`:

```javascript
case 'aggregate-features': {
  const allArgs = args.slice(1);
  const named = parseNamedArgs(allArgs, ['input', 'output', 'mode']);
  const featureAggregator = require('./lib/feature-aggregator.cjs');
  featureAggregator.cmdAggregateFeatures(cwd, {
    inputFile: named.input,
    outputFile: named.output,
    mode: named.mode || 'merge',
  });
  break;
}
```

---

## Data Flow

### End-to-End: brainstorm Output Flowing to PRD to Prototype to Feature Map

```
Developer runs /gsd:brainstorm
         |
         v
brainstorm.md (command)
  Step 0:  Parse flags (--force, --prd-path, --non-interactive)
  Step 1:  Read .planning/PROJECT.md for project context
  Step 2:  Read .planning/FEATURES.md if exists (dedup context for brainstormer)
  Step 3:  Spawn gsd-brainstormer via Task()
           [gsd-brainstormer drives structured conversation]
           [Returns PRD draft as string in Task() response]
  Step 4:  Display PRD draft to user
  Step 5:  AskUserQuestion — approve, iterate, or cancel
  Step 6:  Write approved PRD to .planning/PRD.md (Write tool)
  Step 7:  Auto-run aggregate-features
           node gsd-tools.cjs aggregate-features \
             --input .planning/PRD.md \
             --output .planning/FEATURES.md \
             --mode merge
         |
         v
feature-aggregator.cjs (lib)
  - Parse AC lines from PRD (regex: ^AC-\d+:)
  - Parse feature groups from PRD prose
  - Read existing .planning/FEATURES.md if present
  - Merge: preserve existing entries, add new, flag conflicts
  - Write merged .planning/FEATURES.md
         |
         v
.planning/FEATURES.md updated

Developer runs /gsd:prototype --arch
         |
         v
prototype.md (command) — existing flow with two modifications:
  NEW Step 0b:  Check --arch flag
  NEW Step 2b:  If --arch, inject arch-mode signal into gsd-prototyper Task() context
  Step 4:       Spawn gsd-prototyper with enriched Task() prompt
                [gsd-prototyper reads its context; sees "architecture mode" signal]
                [Builds skeleton-first with stubs + @gsd-todo per component]
  Step 5:       Auto-run extract-tags -> CODE-INVENTORY.md (existing, unchanged)
  NEW Step 5b:  Auto-run aggregate-features --mode merge
                node gsd-tools.cjs aggregate-features \
                  --input .planning/prototype/CODE-INVENTORY.md \
                  --output .planning/FEATURES.md \
                  --mode merge
         |
         v
.planning/FEATURES.md updated with code-derived feature signals
```

### Feature Map: Dual Input Sources

FEATURES.md is written from two input paths that feature-aggregator.cjs must handle:

```
Input Source 1: .planning/PRD.md
  - AC lines (AC-1: ..., AC-2: ...) -> feature entries with PRD provenance
  - Feature groupings from PRD headings/prose
  - Dependency relationships stated explicitly in PRD

Input Source 2: .planning/prototype/CODE-INVENTORY.md
  - @gsd-todo tags   -> implementation intent (what needs building)
  - @gsd-context tags -> architectural decisions (why built this way)
  - @gsd-api tags    -> public contracts (interface surface)
  - @gsd-risk tags   -> identified risks (flag as negative feature signal)

Merge Strategy (in feature-aggregator.cjs):
  - PRD ACs are authoritative sources of truth
  - Code tags refine or confirm PRD features when descriptions match
  - Conflicts (code diverges from PRD intent) -> flag with [DIVERGED] marker in FEATURES.md
  - New code-only features (no matching PRD AC) -> append to "Emerging Features" section
  - Duplicate detection: compare description strings after lowercasing and stop-word removal
```

### Architecture Mode: How gsd-prototyper Behaves Differently

The --arch flag changes only the Task() prompt content sent to the existing gsd-prototyper agent:

```
STANDARD MODE (/gsd:prototype)         ARCHITECTURE MODE (/gsd:prototype --arch)

prototype.md Step 4 Task() includes:  prototype.md Step 4 Task() includes:
  "Build working scaffold code          "Architecture mode: skeleton-first.
   with @gsd-tags embedded..."           Build minimal interface stubs.
                                         Place @gsd-todo on every component.
                                         Do NOT implement business logic.
                                         Place @gsd-context on each module
                                         explaining its architectural role."

gsd-prototyper response:              gsd-prototyper response:
  Implements business logic              Writes interface definitions
  Writes functional code                 Writes empty function bodies
  @gsd-todo for remaining todos          @gsd-todo on every stub
  @gsd-context on key decisions          @gsd-context on every module
```

No changes to gsd-prototyper.md are needed. The behavior difference is entirely driven by what the command injects into the Task() prompt. This is the same mechanism used by prototype.md to inject the AC list.

---

## New vs. Modified Files: Complete Inventory

### New Files (purely additive — no upstream conflicts)

| File | Type | Purpose |
|------|------|---------|
| `commands/gsd/brainstorm.md` | Command descriptor | Entry point for `/gsd:brainstorm` |
| `agents/gsd-brainstormer.md` | Agent descriptor | Conversation-to-PRD agent |
| `agents/gsd-feature-mapper.md` | Agent descriptor | PRD + code tags to FEATURES.md agent |
| `get-shit-done/bin/lib/feature-aggregator.cjs` | Lib module | Structural feature extraction and merge logic |

### Modified Files (minimal, additive changes only)

| File | Modification | Risk Level |
|------|-------------|------------|
| `commands/gsd/prototype.md` | Add Step 0b (parse `--arch` flag); add arch-mode enrichment in Step 4 Task() prompt content; add Step 5b (auto-run aggregate-features after extract-tags) | LOW — additive steps; existing flow unchanged when `--arch` absent |
| `get-shit-done/bin/gsd-tools.cjs` | Add `case 'aggregate-features'` in dispatch switch | LOW — new case only; no existing cases touched |
| `commands/gsd/help.md` or its referenced workflow file | Add `/gsd:brainstorm` to the Code-First command list | LOW — documentation only |
| `get-shit-done/bin/lib/config.cjs` VALID_CONFIG_KEYS | Possibly add `features.auto_update` if feature-map auto-update is made opt-in via config | LOW — new Set entry only |

### Files That Do NOT Need Modification

| File | Reason |
|------|--------|
| `agents/gsd-prototyper.md` | Arch-mode is passed via Task() prompt context, not via agent instructions. The agent reads context injected by the command; no changes to the agent file needed. |
| `get-shit-done/bin/lib/arc-scanner.cjs` | Feature aggregation is a separate concern. arc-scanner's output (CODE-INVENTORY.md) is consumed as input by feature-aggregator; arc-scanner itself is unchanged. |
| `commands/gsd/iterate.md` | The iterate loop is unchanged. It reads CODE-INVENTORY.md and spawns executor. FEATURES.md is a planning artifact outside the iterate loop. |
| `commands/gsd/annotate.md` | No change. The existing auto-chain (annotate -> extract-plan) is complete. Feature map update is downstream of extract-plan, not of annotate itself. |
| All original GSD upstream commands | Zero modifications. Compatibility constraint fully preserved. |
| `bin/install.js` | No explicit registration needed. Installer auto-copies all `agents/gsd-*.md` via wildcard scan. New agents appear automatically after installation. |

---

## Build Order: Dependency Graph

The three v1.2 features have these inter-dependencies:

```
Feature A: /gsd:brainstorm
  Depends on: nothing in existing codebase (new files only)
  Soft dependency on Feature B: brainstorm auto-chains to aggregate-features,
    but brainstorm works without this auto-chain (just doesn't update FEATURES.md)

Feature B: Feature Map (FEATURES.md auto-aggregation)
  Hard depends on: feature-aggregator.cjs lib module
  Hard depends on: aggregate-features case in gsd-tools.cjs
  Auto-chained from: /gsd:brainstorm Step 7 and /gsd:prototype Step 5b
  Independent of: Feature C (architecture mode)

Feature C: Architecture Mode for /gsd:prototype
  Depends on: nothing in Feature A or B
  Fully self-contained: only modifies prototype.md (additive steps)
  Completely independent — can be built and tested in parallel with A and B
```

### Recommended Build Sequence

```
Phase 1 (no dependencies, build first):
  feature-aggregator.cjs lib module
  aggregate-features case in gsd-tools.cjs
  Rationale: Shared infrastructure for Feature B. Build and unit-test
  with node:test before wiring into commands. Purely additive, zero
  breakage risk to existing functionality.

Phase 2a (parallel with 2b, depends only on Phase 1 being committed):
  commands/gsd/brainstorm.md (new command)
  agents/gsd-brainstormer.md (new agent)
  Rationale: Self-contained. Writes PRD.md. Can be tested end-to-end
  without FEATURES.md existing. Add aggregate-features auto-chain
  (Step 7) once Phase 1 is confirmed working.

Phase 2b (parallel with 2a, no dependencies):
  Architecture mode flag in commands/gsd/prototype.md (--arch handling)
  Rationale: Completely independent. Only modifies prototype.md with
  additive steps. Can be built and validated before or after 2a.

Phase 3 (after Phase 1, and after Phase 2a has produced at least one test PRD):
  agents/gsd-feature-mapper.md (new agent)
  FEATURES.md artifact schema definition
  Wire aggregate-features auto-chain into prototype.md Step 5b
  Rationale: Feature mapper needs the aggregate-features tool working.
  Needs a real PRD input from brainstorm (Phase 2a) to validate merge output.

Phase 4 (all three features individually working):
  Integration validation: brainstorm -> PRD.md -> prototype --arch
    -> CODE-INVENTORY.md -> FEATURES.md
  Help command update: add /gsd:brainstorm to Code-First command list
  End-to-end smoke test confirming no regressions in existing commands
```

**Critical path for FEATURES.md:** Phase 1 -> Phase 3.
**Critical path for brainstorm:** Phase 2a (independent).
**Critical path for arch mode:** Phase 2b (independent).

---

## Component Boundaries

### What Belongs in feature-aggregator.cjs (NOT in agents)

The feature aggregator lib module handles deterministic parsing and structural merge operations — pure functions over text input:

- Parse AC lines from a PRD Markdown file (regex matching `^AC-\d+:`)
- Parse relevant @gsd-tags from CODE-INVENTORY.md sections
- Structural merge of two FEATURES.md documents (deduplicate by AC reference and description similarity)
- Detect divergence (code tag present with no matching PRD AC)
- Format merged output as valid FEATURES.md Markdown

This logic is deterministic and must be covered by node:test unit tests. It does NOT belong in gsd-brainstormer or gsd-feature-mapper because LLM agents must not be responsible for precise structural parsing.

### What Belongs in gsd-brainstormer (NOT in the command)

The brainstormer agent handles open-ended conversation and structured synthesis requiring LLM judgment:

- Determine the right clarifying questions given the user's domain and initial description
- Synthesize scattered user answers into coherent problem statement prose
- Infer acceptance criteria from user intent when not stated explicitly
- Detect when the conversation has reached sufficient clarity to produce a PRD draft
- Structure the final PRD draft in a format compatible with prototype.md's AC extraction pattern

This logic is judgment-dependent. It does NOT belong in the command layer.

### What Belongs in gsd-feature-mapper (NOT in feature-aggregator.cjs)

The feature-mapper agent handles semantic synthesis requiring LLM judgment:

- Determine whether two features from different sources (PRD AC and code tag) describe the same capability
- Categorize features into meaningful groups (Auth, Data Pipeline, API Layer, etc.)
- Write the "Why expected / Why valuable" prose for each feature entry
- Identify and articulate genuine conflicts between PRD intent and implementation direction
- Decide whether a code-only feature is an intentional scope expansion or an unintentional drift

These are semantic operations. feature-aggregator.cjs handles structural parsing; gsd-feature-mapper handles synthesis.

---

## Anti-Patterns

### Anti-Pattern 1: Brainstormer Agent Writing PRD.md Directly

**What people do:** Give gsd-brainstormer `Write` tool access and have it write `.planning/PRD.md` directly.

**Why it's wrong:** The user never sees or approves the PRD before it lands on disk. Removes the mandatory human approval gate that is a non-negotiable constraint (see PROJECT.md Out of Scope). Also risks silently overwriting an existing PRD with its AC linkages, orphaning `@gsd-todo(ref:AC-N)` tags in prototype code.

**Do this instead:** gsd-brainstormer returns its PRD draft as a string in the Task() response. The command layer displays it to the user, waits for approval via AskUserQuestion, then writes `.planning/PRD.md` using the Write tool. Agent gets `permissionMode: acceptEdits` but PRD is never committed to disk without user confirmation.

### Anti-Pattern 2: Making Architecture Mode a Separate Agent

**What people do:** Create `gsd-arch-prototyper.md` as a distinct agent parallel to `gsd-prototyper.md`.

**Why it's wrong:** Duplicates the entire prototyper prompt for a behavior difference that amounts to a strategy switch. Two agents to maintain in sync. The existing gsd-arc-executor / gsd-executor split already demonstrates the maintenance cost of this pattern.

**Do this instead:** Inject `"Architecture mode: skeleton-first"` in the Task() prompt from prototype.md when `--arch` is present. gsd-prototyper reads its context and adjusts behavior. One agent, two modes. Zero new files.

### Anti-Pattern 3: Running aggregate-features from Inside an Agent

**What people do:** Have gsd-feature-mapper run `node gsd-tools.cjs aggregate-features` via its own Bash calls to drive its own aggregation pipeline.

**Why it's wrong:** Breaks the command-orchestrator/agent-worker boundary. Agents use Bash only for inspecting project state. Pipeline orchestration (what runs when) belongs in the command layer.

**Do this instead:** The command layer runs `aggregate-features` via Bash, then spawns gsd-feature-mapper via Task() with the aggregate output already prepared as context. The mapper's job is semantic synthesis, not pipeline orchestration.

### Anti-Pattern 4: Auto-Overwriting PRD.md Without User Visibility

**What people do:** `/gsd:brainstorm --force` silently overwrites `.planning/PRD.md` when one already exists.

**Why it's wrong:** Destroys existing PRD and its AC linkages. `@gsd-todo(ref:AC-N)` tags in prototype code become orphaned references. The iterate pipeline loses completeness tracking.

**Do this instead:** When `.planning/PRD.md` exists, default behavior shows the existing PRD and asks: "Append to existing PRD, replace it, or create a separate PRD file?" Only replace on explicit user confirmation. `--force` skips the question but still logs what was replaced in the completion summary.

### Anti-Pattern 5: Coupling Feature Aggregation into arc-scanner.cjs

**What people do:** Add feature map update logic inside `cmdExtractTags` in arc-scanner.cjs so every `extract-tags` call also updates FEATURES.md.

**Why it's wrong:** arc-scanner.cjs has one responsibility: scan files for @gsd-tags and format output. Coupling it to feature map aggregation adds side effects to a pure utility that is called by many commands (iterate.md, prototype.md, annotate.md, extract-plan.md) that have no interest in updating the feature map.

**Do this instead:** Keep arc-scanner.cjs pure. The command layer calls `aggregate-features` as an explicit separate step after `extract-tags` completes, only in commands where that update is appropriate (brainstorm, prototype).

---

## Integration Points: Existing Commands and New Artifacts

Once FEATURES.md and a richer PRD exist, these existing commands can optionally read them for richer context:

| Existing Command | Potential v1.2+ Usage | Required for v1.2 |
|-----------------|----------------------|-------------------|
| `prototype.md` | Reads FEATURES.md for dedup context when spawning gsd-prototyper (avoid rebuilding existing features) | YES — dedup is core brainstorm value |
| `iterate.md` | No change needed — FEATURES.md is a planning artifact outside the iterate loop | No |
| `review-code.md` | Could read FEATURES.md to check implemented features match planned features | No — v1.3+ enhancement |
| `deep-plan.md` | Could read FEATURES.md as context for discuss-phase | No — v1.3+ enhancement |
| `gsd-arc-planner.md` | Could reference FEATURES.md when generating iteration plans | No — v1.3+ enhancement |

For v1.2, prototype.md is the only existing command that reads FEATURES.md. All other integrations are deferred.

---

## Scaling Considerations

This is a single-developer CLI tool. Scaling means running on large codebases and large PRDs without producing garbage output or timing out.

| Scale Concern | Approach |
|---------------|---------|
| PRD with many ACs (>50) | feature-aggregator.cjs warns at 50+ ACs: "Large PRD — consider splitting into milestones." Hard truncation at 200 ACs with notice. Prevents FEATURES.md from becoming unnavigable. |
| Codebase with many @gsd-tags | feature-aggregator.cjs reads CODE-INVENTORY.md (pre-aggregated by arc-scanner), not raw source files. No new scanning performance concern. |
| Existing FEATURES.md with many entries | Merge is O(n*m) where n=existing entries, m=new entries. In practice n < 100. Not a concern for CLI use. |
| Brainstorm conversation depth | gsd-brainstormer should cap at 7 questions maximum (cognitive load research finding). Command enforces by passing `max_questions: 7` in Task() context. |
| Concurrent brainstorm sessions | Not a concern. Single-developer CLI tool. No concurrency model needed. |

---

## Sources

- GSD Code-First codebase direct inspection (2026-03-29):
  - `commands/gsd/prototype.md` — command-orchestrator pattern, Task() prompt enrichment, auto-chain pattern
  - `commands/gsd/iterate.md` — auto-chain pattern, executor dispatch, --non-interactive pattern
  - `commands/gsd/annotate.md` — auto-chain to extract-plan pattern
  - `commands/gsd/review-code.md` — context pre-digestion and injection into Task() prompt
  - `agents/gsd-prototyper.md` — Task() context consumption, project_context reading pattern
  - `agents/gsd-reviewer.md` — two-stage agent with pre-digested context from Task() prompt
  - `get-shit-done/bin/lib/arc-scanner.cjs` — lib module structure, cmdExtractTags pattern, formatAsMarkdown
  - `get-shit-done/bin/lib/config.cjs` — VALID_CONFIG_KEYS pattern, isValidConfigKey
  - `get-shit-done/bin/gsd-tools.cjs` — switch dispatch structure, parseNamedArgs, case extract-tags
  - `bin/install.js` lines 4204-4220 — auto-copy of all `agents/gsd-*.md` via readdirSync wildcard
  - `.planning/config.json` — config schema with arc, workflow, phase_modes namespaces
- `.planning/PROJECT.md` — v1.2 target features, constraints, Out of Scope decisions
- `.planning/research/SUMMARY.md` (v1.1 research) — five-layer architecture description and build order rationale

---

*Architecture research for: GSD Code-First v1.2 Brainstorm and Feature Map*
*Researched: 2026-03-29*
