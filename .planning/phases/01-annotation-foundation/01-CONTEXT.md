# Phase 1: Annotation Foundation - Context

**Gathered:** 2026-03-28
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase delivers the ARC (Annotated Reasoning in Code) annotation standard, the tag scanner tooling in gsd-tools.cjs, the extract-plan command, the gsd-annotator agent for retroactive annotation, and the config schema extension for phase_modes and arc settings. After this phase, developers can annotate code with @gsd-tags and extract structured CODE-INVENTORY.md artifacts.

</domain>

<decisions>
## Implementation Decisions

### Tag Syntax
- **D-01:** Tags use single-line structured format: `@gsd-todo(phase:2) Description text` with parenthesized metadata
- **D-02:** Tags are anchored to comment tokens only (`//`, `#`, `/*`, `--`, `"""`, `'''`) — regex must NOT match @gsd- in strings, URLs, or template literals
- **D-03:** Eight tag types: @gsd-context, @gsd-decision, @gsd-todo, @gsd-constraint, @gsd-pattern, @gsd-ref, @gsd-risk, @gsd-api
- **D-04:** Metadata in parentheses is optional — `@gsd-todo Fix this` is valid, `@gsd-todo(phase:2, priority:high) Fix this` adds structured metadata

### Scanner Architecture
- **D-05:** Tag scanner implemented as new lib module `get-shit-done/bin/lib/arc-scanner.cjs` following the existing module pattern (like config.cjs, phase.cjs)
- **D-06:** Scanner exposed via gsd-tools.cjs as subcommand: `gsd-tools.cjs extract-tags [--phase N] [--type TYPE] [--format md|json]`
- **D-07:** Scanner uses Node.js built-in `RegExp` with multiline flags — zero runtime dependencies, matching the zero-dep constraint
- **D-08:** Scanner output formats: JSON (for agent consumption) and Markdown (for CODE-INVENTORY.md)

### Extract Plan Command
- **D-09:** extract-plan command defined in `commands/gsd/extract-plan.md` — calls `gsd-tools.cjs extract-tags`, writes `.planning/prototype/CODE-INVENTORY.md`, shows terminal summary
- **D-10:** CODE-INVENTORY.md groups tags by type, then by file, with phase reference cross-links and summary statistics

### Annotator Agent
- **D-11:** gsd-annotator agent defined in `agents/gsd-annotator.md` — operates at directory scope with file glob filtering
- **D-12:** annotate command defined in `commands/gsd/annotate.md` — spawns gsd-annotator then auto-runs extract-plan on completion
- **D-13:** Annotator reads existing code + PROJECT.md + REQUIREMENTS.md to determine appropriate annotations

### Config Schema
- **D-14:** New top-level keys `arc` and `phase_modes` added to `.planning/config.json` schema
- **D-15:** `arc` config section: `{ enabled: boolean, tag_prefix: "@gsd-", comment_anchors: ["//", "#", "/*", "--"] }`
- **D-16:** `phase_modes` config section: maps phase numbers to mode strings (`code-first`, `plan-first`, `hybrid`)
- **D-17:** Config validation extended in `get-shit-done/bin/lib/config.cjs`

### Claude's Discretion
- Regex pattern specifics for edge case handling (strings, template literals)
- CODE-INVENTORY.md exact formatting and section ordering
- arc-standard.md language examples beyond the core 4 (JS, Python, Go, Rust)
- Test fixture design for false-positive prevention

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Architecture
- `get-shit-done/bin/gsd-tools.cjs` — Main CLI tool; new `extract-tags` subcommand goes here
- `get-shit-done/bin/lib/config.cjs` — Config schema validation; extend for arc and phase_modes
- `get-shit-done/bin/lib/phase.cjs` — Phase operations; pattern for new lib module
- `get-shit-done/bin/lib/core.cjs` — Core utilities; follow patterns for file I/O

### Agent and Command Patterns
- `agents/gsd-executor.md` — Agent prompt pattern to follow for gsd-annotator
- `commands/gsd/quick.md` — Lightweight command pattern most similar to extract-plan
- `commands/gsd/fast.md` — Inline execution pattern reference

### Research
- `.planning/research/STACK.md` — Zero-dep constraint, CJS pattern, node:test requirement
- `.planning/research/PITFALLS.md` — Regex false positive risks, fork divergence policy
- `.planning/research/ARCHITECTURE.md` — Component boundaries, data flow, gsd-tools gateway pattern

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `get-shit-done/bin/lib/*.cjs` — 17 existing lib modules provide exact pattern for arc-scanner.cjs
- `get-shit-done/bin/gsd-tools.cjs` — Command dispatch pattern for adding `extract-tags` subcommand
- `agents/*.md` — 18 existing agent definitions provide YAML frontmatter + prompt template

### Established Patterns
- All lib modules use CJS (`module.exports`, `require()`)
- All tests use `node:test` built-in test runner
- Config operations route through `lib/config.cjs` → `gsd-tools.cjs`
- Agent prompts are Markdown with YAML frontmatter for Claude Code registration

### Integration Points
- `gsd-tools.cjs` command dispatch — new `extract-tags` case in the main switch
- `config.cjs` schema validation — new `arc` and `phase_modes` fields
- `bin/install.js` — must be updated in Phase 3 to copy new files (not this phase)

</code_context>

<specifics>
## Specific Ideas

- ARC standard must stabilize before any real codebase is annotated (from Pitfalls research — standard changes force re-annotation)
- Upstream files are read-only policy: do NOT modify existing agent/command files in this phase
- Tag scanner must be tested against edge-case fixtures from day one (strings containing @gsd-, URLs, template literals)

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-annotation-foundation*
*Context gathered: 2026-03-28*
