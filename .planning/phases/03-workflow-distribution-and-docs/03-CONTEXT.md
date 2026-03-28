# Phase 3: Workflow, Distribution, and Docs - Context

**Gathered:** 2026-03-28
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase delivers the complete code-first workflow as an installable npm package. It wires the iterate loop (extract-tags -> code-planner -> approval -> executor), adds mode configuration commands (set-mode, deep-plan), updates the installer to copy all new agent and command files with GSD_CF_ namespaced markers, and produces user documentation (README, help command, user guide). After this phase, `npx gsd-code-first@latest` installs a fully functional code-first fork with documentation.

</domain>

<decisions>
## Implementation Decisions

### Iterate Command
- **D-01:** `iterate` command defined in `commands/gsd/iterate.md` -- orchestrates the full code-first loop: extract-tags -> gsd-code-planner -> approval gate -> executor
- **D-02:** Default behavior: interactive approval gate that pauses to show the generated plan and waits for user approval/rejection before execution
- **D-03:** `--non-interactive` flag auto-approves the plan for CI/headless pipelines (per ITER-03)
- **D-04:** `--verify` flag runs verification after executor completes (per ITER-02)
- **D-05:** `--annotate` flag runs gsd-annotator to refresh @gsd-tags after executor completes (per ITER-02)
- **D-06:** iterate chains steps sequentially: each step's output feeds the next. If any step fails, iterate stops and reports the failure.

### Mode Configuration
- **D-07:** `set-mode` command defined in `commands/gsd/set-mode.md` -- writes to config.json `default_phase_mode` or `phase_modes[N]` (per MODE-01)
- **D-08:** Accepts `code-first`, `plan-first`, or `hybrid` as mode values
- **D-09:** Per-phase override: `set-mode code-first --phase 3` sets mode for a specific phase
- **D-10:** Active mode is visible at command startup -- gsd-tools.cjs shows current mode in status output

### Deep-Plan Command
- **D-11:** `deep-plan` command defined in `commands/gsd/deep-plan.md` -- chains discuss-phase then plan-phase for phases needing upfront reasoning (per MODE-03)
- **D-12:** Passes through phase number and any flags to both commands

### Installer Updates
- **D-13:** `bin/install.js` updated to copy all new agent files: gsd-prototyper.md, gsd-code-planner.md, gsd-arc-executor.md, gsd-arc-planner.md
- **D-14:** `bin/install.js` updated to copy all new command files: prototype.md, iterate.md, set-mode.md, deep-plan.md
- **D-15:** Installer markers use `GSD_CF_` namespace prefix (e.g., `GSD_CF_PROTOTYPER`, `GSD_CF_CODE_PLANNER`) to avoid conflicts with upstream GSD installations (per DIST-03)
- **D-16:** `package.json` already has name `gsd-code-first` and bin entry `get-shit-done-cc` pointing to `bin/install.js` -- verify and update version if needed (per DIST-02)

### Help Command
- **D-17:** Help output updated to list all new commands: prototype, iterate, annotate, extract-plan, set-mode, deep-plan (per DOCS-01)
- **D-18:** Each command entry includes a one-line description

### Documentation
- **D-19:** README.md documents installation (`npx gsd-code-first@latest`), the code-first workflow, and quick-start examples (per DOCS-02)
- **D-20:** User guide (separate section in README or standalone file) explains ARC tags, prototype -> iterate workflow, mode switching, and when to use code-first vs plan-first vs hybrid (per DOCS-03)
- **D-21:** Documentation references arc-standard.md for detailed tag syntax rather than duplicating it

### Claude's Discretion
- Exact step-by-step orchestration within iterate command (how to chain agent spawns)
- set-mode command output formatting
- deep-plan flag passthrough implementation
- README.md structure and section ordering
- User guide depth and examples
- How help command discovers and lists available commands

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Installer and Distribution
- `bin/install.js` -- Current installer. Must be updated to copy new agent and command files with GSD_CF_ markers.
- `package.json` -- Package metadata. Already named gsd-code-first with bin entry.
- `scripts/build-hooks.js` -- Pre-publish hook validation pattern.

### Existing Command Patterns
- `commands/gsd/annotate.md` -- Agent-spawning command with auto-chain. Pattern for iterate command.
- `commands/gsd/prototype.md` -- Phase 2 command. Pattern for iterate (similar agent spawn + chain).
- `commands/gsd/extract-plan.md` -- Phase 1 command. Called within iterate loop.
- `commands/gsd/discuss-phase.md` -- Existing command. deep-plan wraps this + plan-phase.

### Existing Agent Patterns
- `agents/gsd-code-planner.md` -- Phase 2 agent. Spawned by iterate command.
- `agents/gsd-arc-executor.md` -- Phase 2 wrapper agent. Spawned by iterate when arc.enabled.
- `agents/gsd-annotator.md` -- Phase 1 agent. Spawned by iterate with --annotate flag.

### Config and Tools
- `get-shit-done/bin/lib/config.cjs` -- Config schema with arc and phase_modes. set-mode writes here.
- `get-shit-done/bin/gsd-tools.cjs` -- CLI gateway. 63 case branches. set-mode subcommand goes here.
- `get-shit-done/references/arc-standard.md` -- ARC tag reference. Documentation references this.

### Research (from Phase 1)
- `.planning/research/STACK.md` -- Zero-dep constraint, CJS pattern, node:test requirement.
- `.planning/research/ARCHITECTURE.md` -- Component boundaries, gsd-tools gateway pattern.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `commands/gsd/annotate.md` -- Direct template for iterate command (agent spawn + auto-chain pattern)
- `commands/gsd/prototype.md` -- Direct template for iterate command (Task spawn with $ARGUMENTS passthrough)
- `bin/install.js` -- Existing installer with agent/command copy loops and marker system
- `get-shit-done/bin/gsd-tools.cjs` -- CLI gateway with parseNamedArgs() helper for flag parsing
- `get-shit-done/bin/lib/config.cjs` -- Config operations with validation. set-mode uses config-set.

### Established Patterns
- Command files: YAML frontmatter + objective + process sections
- Agent spawning: Task tool with subagent_type parameter
- Auto-chaining: bash command to run extract-tags after agent completes
- Config writes: `gsd-tools.cjs config-set key value` pattern
- Flag parsing: `parseNamedArgs()` in gsd-tools.cjs
- Installer markers: `GSD_MARKER_` prefix pattern (fork uses `GSD_CF_`)

### Integration Points
- `commands/gsd/iterate.md` -- New command file
- `commands/gsd/set-mode.md` -- New command file
- `commands/gsd/deep-plan.md` -- New command file
- `bin/install.js` -- Modified to add new file copies
- `get-shit-done/bin/gsd-tools.cjs` -- Modified to add set-mode subcommand and mode display
- Help command output -- Modified to list new commands

</code_context>

<specifics>
## Specific Ideas

- package.json already has name `gsd-code-first` and version `2.0.0-alpha.1` -- installer update is additive
- 63 existing case branches in gsd-tools.cjs -- set-mode adds one more
- Iterate is the flagship command of the fork -- it demonstrates the full code-first value proposition
- The approval gate in iterate must be the ONLY place execution is authorized -- no auto-execute without explicit opt-in
- GSD_CF_ marker namespace keeps the fork's installer from conflicting with upstream `get-shit-done-cc` installations

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 03-workflow-distribution-and-docs*
*Context gathered: 2026-03-28*
