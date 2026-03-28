# Architecture Patterns

**Domain:** Node.js CLI framework with AI agent orchestration, code annotation parsing, and extensible command system
**Researched:** 2026-03-28

## Existing System Architecture (GSD Upstream)

Before describing the Code-First additions, the upstream architecture must be understood — all new components are additive to this foundation.

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
│  - gsd-debugger    │   │  ├── lib/config.cjs                │
│  - ... (13 total)  │   │  ├── lib/init.cjs                  │
└────────────────────┘   │  ├── lib/roadmap.cjs               │
                         │  ├── lib/template.cjs              │
                         │  ├── lib/verify.cjs                │
                         │  └── lib/model-profiles.cjs        │
                         └──────────────────┬─────────────────┘
                                            │ reads/writes
                                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Planning State Layer                                        │
│  .planning/                                                  │
│  ├── config.json        (project config + feature flags)    │
│  ├── STATE.md           (current position, decisions)       │
│  ├── ROADMAP.md         (phase definitions)                 │
│  ├── REQUIREMENTS.md    (tracked requirements)              │
│  ├── phase-N-*/         (per-phase artifacts)               │
│  │   ├── PLAN.md        (execution plan)                    │
│  │   ├── SUMMARY.md     (completion record)                 │
│  │   └── CONTEXT.md     (phase context)                     │
│  └── research/          (research artifacts)                │
└─────────────────────────────────────────────────────────────┘
```

### Key Architectural Properties of the Existing System

**Command descriptor pattern.** Every slash-command (`/gsd:execute-phase`) is a Markdown file with YAML frontmatter (`name`, `description`, `allowed-tools`). The command file contains no logic — it injects workflow files via `@`-references and passes `$ARGUMENTS` through. Logic lives entirely in the workflow layer.

**Orchestrator-subagent pattern.** Workflow files are lean orchestrators. They call `gsd-tools.cjs init <workflow>` to load all context as JSON, then spawn typed subagents (`gsd-executor`, `gsd-planner`, etc.) via the `Task()` API. Each subagent runs independently with a fresh context window. The orchestrator collects results and advances state.

**Parallel wave execution.** Plans within a phase are grouped into dependency waves. Each wave's plans execute in parallel (one subagent per plan). This is the primary performance mechanism.

**Typed agents with model profiles.** Each agent type has a model assignment per profile tier (quality/balanced/budget). `gsd-tools.cjs resolve-model <agent-type>` resolves the correct model at runtime. Model selection is config-driven, not hardcoded.

**gsd-tools.cjs as the single state interface.** All reads and writes to `.planning/` go through `gsd-tools.cjs`. Workflows and agents never directly manipulate planning files — they call the CLI tool. This is the system's state boundary.

**Additive installer.** `bin/install.js` copies commands, agents, and tool binaries into the user's Claude config directory at `~/.claude/`. New features extend this installer with new files; nothing is modified in place.

---

## Code-First Extension Architecture

The ARC (Annotated Reasoning in Code) system adds four new component types to the existing architecture. All are strictly additive.

### New Component Map

```
User Code (any language)
      │  contains
      ▼
┌─────────────────────────────────────────────────────────────┐
│  ARC Annotation Layer (NEW)                                  │
│  Structured @gsd-tags embedded in code comments             │
│  - @gsd-context   (what this code does / why)               │
│  - @gsd-decision  (architectural choice recorded)           │
│  - @gsd-todo      (work remaining)                          │
│  - @gsd-constraint (limits/requirements imposed)            │
│  - @gsd-pattern   (reusable pattern identified)             │
│  - @gsd-ref       (cross-reference to other code/docs)      │
│  - @gsd-risk      (technical/product risk flagged)          │
│  - @gsd-api       (public interface specification)          │
└──────────────────────────┬──────────────────────────────────┘
                           │ read by
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Tag Scanner (NEW — extends gsd-tools.cjs)                  │
│  Regex-based extraction of @gsd-tags from any text file     │
│  - Input: file path(s) or directory                         │
│  - Output: structured JSON (tag type, value, file, line)    │
│  - Language-agnostic (works on JS, TS, Python, etc.)        │
│  - New gsd-tools.cjs commands:                              │
│      scan-tags <path>     → JSON array of extracted tags    │
│      extract-plan <path>  → generate CODE-INVENTORY.md      │
└──────────────────────────┬──────────────────────────────────┘
                           │ produces
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  CODE-INVENTORY.md (NEW planning artifact)                  │
│  Lives in .planning/ (alongside STATE.md, ROADMAP.md)       │
│  - Grouped by tag type                                      │
│  - Linked to source file and line                           │
│  - Input consumed by code-planner agent                     │
└──────────────────────────┬──────────────────────────────────┘
                           │ consumed by
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  New Agent Layer (NEW — extends agents/*.md)                │
│                                                              │
│  gsd-prototyper    — builds prototypes with ARC annotations │
│  gsd-code-planner  — reads code + tags → planning artifacts │
│  gsd-annotator     — retroactively annotates existing code  │
│                                                              │
│  Modified agents:                                            │
│  gsd-executor      — now required to add ARC comments       │
│  gsd-planner       — now reads CODE-INVENTORY.md as input   │
└──────────────────────────┬──────────────────────────────────┘
                           │ spawned by
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  New Command Layer (NEW — extends commands/gsd/*.md)        │
│                                                              │
│  /gsd:prototype    → spawns gsd-prototyper                  │
│  /gsd:annotate     → spawns gsd-annotator + extract-plan    │
│  /gsd:extract-plan → calls gsd-tools scan-tags → inventory  │
│  /gsd:iterate      → extract-tags → code-planner → approve  │
│                       → executor pipeline                    │
│  /gsd:deep-plan    → wraps discuss-phase + plan-phase        │
│  /gsd:set-mode     → writes phase_modes config              │
│                                                              │
│  Modified commands:                                          │
│  (none — upstream commands unchanged)                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Boundaries

| Component | Responsibility | Reads From | Writes To |
|-----------|---------------|------------|-----------|
| ARC annotations (in user code) | Embed structured planning metadata | — | User source files |
| Tag scanner (gsd-tools.cjs extension) | Regex-extract @gsd-tags, output JSON | User source files | stdout (JSON) |
| `extract-plan` command | Orchestrate scan → inventory generation | Tag scanner output | `.planning/CODE-INVENTORY.md` |
| `CODE-INVENTORY.md` | Structured registry of all tags in codebase | — | Consumed by code-planner agent |
| `gsd-prototyper` agent | Build prototype code with ARC annotations | Project context, ROADMAP.md | User source files with @gsd-tags |
| `gsd-code-planner` agent | Generate planning artifacts from code | CODE-INVENTORY.md, source files | PLAN.md, SUMMARY.md, ROADMAP.md updates |
| `gsd-annotator` agent | Add ARC annotations to existing code | User source files | User source files with @gsd-tags added |
| Modified `gsd-executor` | Execute plans + add ARC comments during execution | PLAN.md | User source files (with @gsd-tags), SUMMARY.md |
| `config.json` extension | Store `phase_modes`, `arc` settings, `default_phase_mode` | — | `.planning/config.json` |
| `bin/install.js` extension | Install new agents and commands | Package files | `~/.claude/` |

---

## Data Flow

### Code-First Path (new workflow)

```
Developer writes prototype code
        │
        ▼
Code contains @gsd-tags in comments
        │
        ▼
/gsd:extract-plan
  → gsd-tools scan-tags ./src   [regex scan, outputs JSON]
  → gsd-tools extract-plan      [JSON → CODE-INVENTORY.md]
        │
        ▼
.planning/CODE-INVENTORY.md exists
        │
        ▼
/gsd:iterate
  Phase 1: gsd-code-planner reads CODE-INVENTORY.md + source
           → generates PLAN.md for each identified work unit
  Phase 2: User reviews and approves plans (approval gate)
  Phase 3: gsd-executor executes approved plans
           → executor adds new @gsd-tags as it works
           → cycle repeats
```

### Prototype Path (new workflow)

```
/gsd:prototype "build auth system"
  → gsd-prototyper reads project context
  → writes prototype code with @gsd-tags embedded
  → auto-runs /gsd:extract-plan
  → produces CODE-INVENTORY.md
```

### Retroactive Annotation Path (new workflow)

```
Existing codebase (no @gsd-tags)
        │
        ▼
/gsd:annotate
  → gsd-annotator reads source files
  → adds @gsd-tags inline as comments
  → runs gsd-tools extract-plan
  → produces CODE-INVENTORY.md
```

### Plan-First Path (unchanged upstream path)

```
/gsd:discuss-phase → /gsd:plan-phase → /gsd:execute-phase
```

Both paths produce the same downstream artifacts (PLAN.md, SUMMARY.md, STATE.md updates). The code-first path just replaces the discuss/plan steps with annotation extraction.

---

## Suggested Build Order

Dependencies flow upward: later components depend on earlier ones. Build bottom-up.

### Layer 1: Foundation (no dependencies on new code)

**1a. ARC annotation standard**
Define the @gsd-tag vocabulary and syntax rules in documentation. This is specification work, no code. All other components depend on this being stable.

**1b. Config schema extension**
Add `phase_modes`, `arc`, and `default_phase_mode` keys to the config schema in `lib/config.cjs`. These unlock mode-switching for later commands. Low risk, touches existing code minimally.

**1c. Tag scanner in gsd-tools.cjs**
Add `scan-tags` and `extract-plan` commands to `gsd-tools.cjs`. Regex extraction, no external dependencies. Produces JSON output — testable in isolation before any agent uses it.

### Layer 2: Planning Artifact (depends on Layer 1)

**2a. CODE-INVENTORY.md format**
Finalize the schema of `CODE-INVENTORY.md` (sections, link format, tag grouping). This is consumed by the planner agent — if the schema changes later, the agent prompt changes too. Lock it early.

**2b. extract-plan command**
Thin command file that calls `gsd-tools extract-plan`. Depends on tag scanner (1c) and inventory format (2a). Very little logic.

### Layer 3: Agents (depends on Layers 1-2)

**3a. gsd-annotator**
Reads source files, writes @gsd-tags. No dependency on CODE-INVENTORY.md — it produces input for the scanner. Can be built independently of gsd-code-planner.

**3b. gsd-prototyper**
Writes prototype code with ARC annotations baked in. Depends on ARC standard (1a) and being able to call extract-plan afterward.

**3c. gsd-code-planner**
Reads CODE-INVENTORY.md and source files, produces PLAN.md. This is the most complex new agent — depends on inventory format (2a) being stable.

**3d. Modified gsd-executor**
Add ARC comment obligation to existing executor prompt. Minimal change — existing behavior preserved, new requirement added.

**3e. Modified gsd-planner**
Add code-based planning mode that reads CODE-INVENTORY.md as input. Feature-flagged by `phase_modes` config.

### Layer 4: Command Orchestration (depends on Layer 3)

**4a. annotate command**
Spawns gsd-annotator (3a), then runs extract-plan. Depends on both.

**4b. prototype command**
Spawns gsd-prototyper (3b). Straightforward.

**4c. iterate command**
Orchestrates: extract-tags → gsd-code-planner → approval gate → gsd-executor. Depends on all agents in Layer 3. Most complex command — build last.

**4d. deep-plan command**
Wraps existing discuss-phase + plan-phase. Low complexity — primarily composition.

**4e. set-mode command**
Writes to config.json `phase_modes`. Depends on config schema extension (1b).

### Layer 5: Installer and Distribution (depends on all above)

**5a. Updated bin/install.js**
Extend installer to copy new agents and commands. Touch only after all agent and command files are finalized.

**5b. package.json and README**
Update name, docs. Final step.

---

## Build Order Summary Table

| Layer | Component | Depends On | Risk |
|-------|-----------|-----------|------|
| 1a | ARC annotation standard | — | Low |
| 1b | Config schema extension | — | Low |
| 1c | Tag scanner (gsd-tools.cjs) | 1a | Medium |
| 2a | CODE-INVENTORY.md format | 1a, 1c | Low |
| 2b | extract-plan command | 1c, 2a | Low |
| 3a | gsd-annotator agent | 1a | Medium |
| 3b | gsd-prototyper agent | 1a, 2b | Medium |
| 3c | gsd-code-planner agent | 2a | High |
| 3d | Modified gsd-executor | 1a | Low |
| 3e | Modified gsd-planner | 1b, 2a | Medium |
| 4a | annotate command | 3a, 2b | Low |
| 4b | prototype command | 3b | Low |
| 4c | iterate command | 3c, 3d, 3e | High |
| 4d | deep-plan command | existing commands | Low |
| 4e | set-mode command | 1b | Low |
| 5a | Updated installer | all above | Low |
| 5b | package.json + README | all above | Low |

---

## Patterns to Follow

### Pattern: Command as thin descriptor
Commands contain no logic. The command file injects a workflow file via `@`-reference and passes arguments. All logic lives in the workflow or gsd-tools.cjs. New commands (prototype, annotate, iterate, extract-plan, deep-plan, set-mode) must follow this pattern.

### Pattern: gsd-tools.cjs as state gateway
Nothing reads or writes `.planning/` directly. All state access is through gsd-tools.cjs subcommands. The tag scanner and inventory generator must write through gsd-tools.cjs, not via direct `fs.writeFileSync` calls in agent prompts.

### Pattern: Additive-only agent modifications
The `gsd-executor` and `gsd-planner` modifications must add behavior without removing existing behavior. Use feature flags (`phase_modes` config) to gate new behavior. A phase in "plan-first" mode must produce identical results to the upstream executor.

### Pattern: init command for compound context
Complex orchestration workflows use `gsd-tools init <workflow>` to load all context as a single JSON blob before spawning agents. The `iterate` command should follow this pattern — create an `init iterate` compound command in `lib/init.cjs` that bundles CODE-INVENTORY.md content, config, and phase state.

### Pattern: Typed subagents with exact names
When spawning subagents, use the exact agent type name (`gsd-prototyper`, `gsd-code-planner`, `gsd-annotator`). These names must match the agent file names in `agents/` exactly. Never fall back to `general-purpose`.

---

## Anti-Patterns to Avoid

### Anti-Pattern: Embedding logic in command files
Command files that contain conditional logic, loops, or state reads directly are hard to test and diverge from the existing architecture. All logic belongs in workflows or gsd-tools.cjs.

**Instead:** Keep command files as descriptor + workflow reference. Put logic in workflow files.

### Anti-Pattern: Direct .planning/ file manipulation in agent prompts
Agents that call `echo "..." > .planning/CODE-INVENTORY.md` directly bypass the state gateway. This breaks consistency guarantees and makes testing harder.

**Instead:** Route all writes through `gsd-tools.cjs` commands. Add a `write-inventory` subcommand if needed.

### Anti-Pattern: AST-based tag extraction
Full AST parsing is language-specific, adds dependencies, and is brittle across language versions. The ARC tag format is designed to work with regex.

**Instead:** Use regex patterns anchored to `@gsd-<tagname>` with simple key-value or freeform value extraction. Language-agnostic, zero dependencies.

### Anti-Pattern: Tight coupling between code-first and plan-first paths
If the `iterate` command assumes CODE-INVENTORY.md always exists, it will fail for phases that started on the plan-first path. The two paths should produce compatible intermediate artifacts.

**Instead:** The `gsd-code-planner` agent should gracefully handle missing CODE-INVENTORY.md by triggering `extract-plan` first, or by reading source files directly.

### Anti-Pattern: Modifying upstream command files
The upstream commands (`discuss-phase`, `plan-phase`, `execute-phase`, etc.) must not be changed. Any modification risks merge conflicts with upstream GSD updates and breaks the compatibility guarantee.

**Instead:** New functionality goes in new files. Where behavior must change (executor ARC obligation), make changes purely additive or use config flags to gate them.

---

## Scalability Considerations

This is a local CLI tool. "Scale" here means handling large codebases and long iteration cycles, not distributed throughput.

| Concern | Small codebase (<5K LOC) | Medium (5K-50K LOC) | Large (>50K LOC) |
|---------|--------------------------|---------------------|-----------------|
| Tag scanning performance | Instant (regex) | Fast (<1s) | Needs directory filtering; scan only changed files |
| CODE-INVENTORY.md size | Small, fits in context window | Medium, fits in context | May need pagination; agent reads sections not full file |
| Agent context budget | All tags fit | Most tags fit | Tag scanner must support filtering by type or directory |
| iterate pipeline cost | Low | Moderate | High; use `--wave N` style phasing and incremental scans |

For v1 with regex-based extraction: scanning is bounded by file I/O speed, not computation. No performance concerns until codebases exceed ~200K LOC.

---

## Sources

- Upstream GSD codebase: `get-shit-done/bin/gsd-tools.cjs`, `get-shit-done/workflows/execute-phase.md`, `agents/*.md` (directly inspected, HIGH confidence)
- Orchestrator-subagent pattern: `get-shit-done/workflows/execute-phase.md` core_principle and runtime_compatibility sections (HIGH confidence)
- Model profile routing: `get-shit-done/bin/lib/model-profiles.cjs` (HIGH confidence)
- State gateway pattern: `lib/core.cjs`, `lib/init.cjs`, `lib/config.cjs` (HIGH confidence)
- AI agent orchestration patterns: [Azure Architecture Center — AI Agent Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) (MEDIUM confidence)
- CLI plugin architecture: [oclif — The Open CLI Framework](https://oclif.io/) (MEDIUM confidence, for pattern comparison only — GSD does not use oclif)
- Code annotation extraction: [openedx/code-annotations](https://github.com/openedx/code-annotations) (MEDIUM confidence, reference for regex-over-AST rationale)
