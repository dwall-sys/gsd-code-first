# Phase 2: Core Agents - Context

**Gathered:** 2026-03-28
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase delivers the three new agents that power code-first development: gsd-prototyper (builds annotated prototypes from scratch), gsd-code-planner (reads annotations to produce plans), and modified executor/planner behavior (ARC comment obligations during execution). It also delivers the `prototype` command, the PROTOTYPE-LOG.md template, and config-gated agent modifications. After this phase, developers can run `prototype` to generate annotated code and have gsd-code-planner produce plans from those annotations.

</domain>

<decisions>
## Implementation Decisions

### Prototyper Agent
- **D-01:** gsd-prototyper agent defined in `agents/gsd-prototyper.md` — reads PROJECT.md, REQUIREMENTS.md, and ROADMAP.md as context, scopes work to phases specified via `--phases` flag
- **D-02:** Prototyper produces working scaffold code (runnable, demonstrates structure and intent) with @gsd-tags already embedded — not production-ready implementations
- **D-03:** Prototyper follows ARC standard from `get-shit-done/references/arc-standard.md` for all tag placement and syntax
- **D-04:** Prototyper auto-runs `extract-plan` on completion to generate CODE-INVENTORY.md from the annotated prototype

### Prototype Command
- **D-05:** `prototype` command defined in `commands/gsd/prototype.md` — spawns gsd-prototyper with project context files
- **D-06:** Command supports `--phases N` flag for scoping to specific phase(s)
- **D-07:** PROTOTYPE-LOG.md template captures: what was built, decisions made during prototyping, and open @gsd-todos — written by prototyper on completion

### Code-Planner Agent
- **D-08:** gsd-code-planner agent defined in `agents/gsd-code-planner.md` — reads CODE-INVENTORY.md as primary input AND scans source @gsd-tags as supplementary input
- **D-09:** Code-planner generates compact Markdown PLAN.md files: tasks, target files, and success criteria — no XML wrappers, no `<research>` sections, no plan-check blocks
- **D-10:** Code-planner plans are compact enough for a single executor pass — no multi-plan phase decomposition unless the annotation scope demands it

### Agent Modifications (Wrapper Strategy)
- **D-11:** gsd-executor modification implemented as NEW agent `agents/gsd-arc-executor.md` — a wrapper that extends gsd-executor behavior with ARC comment obligations (adds @gsd-decision tags, removes completed @gsd-todo tags)
- **D-12:** gsd-planner modification implemented as NEW agent `agents/gsd-arc-planner.md` — a wrapper that extends gsd-planner to accept @gsd-tags as planning input alongside or instead of requirements docs
- **D-13:** Wrapper agents are the ONLY approach — upstream agent files (gsd-executor.md, gsd-planner.md) remain unmodified for merge compatibility

### Config Gating
- **D-14:** Agent modifications gated by `arc.enabled` in `.planning/config.json` — when false, wrapper agents behave identically to originals
- **D-15:** Workflow commands check `config.arc.enabled` and `config.default_phase_mode` to determine which agent variant to spawn

### Claude's Discretion
- Exact prompt structure and instruction ordering within agent .md files
- How the code-planner discovers and reads relevant source files beyond CODE-INVENTORY.md
- PROTOTYPE-LOG.md exact Markdown template layout and section ordering
- How wrapper agents delegate to or extend the behavior of upstream agents (instruction inclusion vs reference)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### ARC Standard
- `get-shit-done/references/arc-standard.md` -- Tag types, syntax rules, comment anchor rules, metadata keys. Authoritative spec for all annotation behavior.

### Existing Agent Patterns
- `agents/gsd-executor.md` -- Upstream executor agent. Wrapper gsd-arc-executor.md must extend this behavior, not replace it.
- `agents/gsd-planner.md` -- Upstream planner agent. Wrapper gsd-arc-planner.md must extend this behavior, not replace it.
- `agents/gsd-annotator.md` -- Phase 1 agent. Pattern for ARC-aware agent design (reads arc-standard.md, PROJECT.md, REQUIREMENTS.md).

### Existing Command Patterns
- `commands/gsd/annotate.md` -- Phase 1 command. Pattern for spawning an agent + auto-chaining to extract-plan.
- `commands/gsd/extract-plan.md` -- Phase 1 command. Called by prototype command on completion.

### Scanner and Config
- `get-shit-done/bin/lib/arc-scanner.cjs` -- Tag scanner module. Code-planner may call extract-tags for fresh scans.
- `get-shit-done/bin/lib/config.cjs` -- Config schema with arc and phase_modes sections. Agent gating reads from here.
- `get-shit-done/bin/gsd-tools.cjs` -- CLI gateway. extract-tags subcommand already wired.

### Research (from Phase 1)
- `.planning/research/STACK.md` -- Zero-dep constraint, CJS pattern, node:test requirement
- `.planning/research/PITFALLS.md` -- Regex false positive risks, fork divergence policy
- `.planning/research/ARCHITECTURE.md` -- Component boundaries, data flow, gsd-tools gateway pattern

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `agents/gsd-annotator.md` -- ARC-aware agent pattern (reads arc-standard.md, PROJECT.md, REQUIREMENTS.md). Direct template for gsd-prototyper.
- `commands/gsd/annotate.md` -- Agent-spawning command with extract-plan auto-chain. Direct template for prototype command.
- `agents/gsd-executor.md` -- Execution agent with atomic commits and state management. Base for gsd-arc-executor wrapper.
- `agents/gsd-planner.md` -- Planning agent with PLAN.md generation. Base for gsd-arc-planner wrapper.
- `get-shit-done/bin/lib/arc-scanner.cjs` -- Tag scanner with JSON/Markdown output. Code-planner uses this.

### Established Patterns
- Agent files: YAML frontmatter (name, description, tools, color) + Markdown prompt body
- Command files: YAML frontmatter (name, description, argument-hint, allowed-tools) + objective + process
- All agents read CLAUDE.md for project-specific conventions
- Config operations route through lib/config.cjs
- Tests use node:test with assertions from node:assert

### Integration Points
- `commands/gsd/prototype.md` -- New command file, follows annotate.md pattern
- `agents/gsd-prototyper.md` -- New agent file, follows gsd-annotator.md pattern
- `agents/gsd-code-planner.md` -- New agent file, reads CODE-INVENTORY.md + source tags
- `agents/gsd-arc-executor.md` -- Wrapper agent extending gsd-executor behavior
- `agents/gsd-arc-planner.md` -- Wrapper agent extending gsd-planner behavior
- `bin/install.js` -- Must be updated in Phase 3 (not this phase) to copy new files

</code_context>

<specifics>
## Specific Ideas

- STATE.md decision: "gsd-executor modification to be implemented as new gsd-arc-executor.md wrapper, not a patch to upstream file" -- this is a locked decision from Phase 1 research
- STATE.md concern: "gsd-code-planner: HIGH risk agent -- prompt structure for reliable PLAN.md from CODE-INVENTORY.md is not fully resolved. Consider /gsd:research-phase before planning Phase 2"
- The wrapper agent approach preserves upstream mergeability -- the core project constraint
- Prototyper follows the same annotator pattern: read context docs, operate on files, auto-chain to extract-plan

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 02-core-agents*
*Context gathered: 2026-03-28*
