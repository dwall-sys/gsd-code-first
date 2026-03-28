# Project Research Summary

**Project:** GSD Code-First Fork
**Domain:** Code-annotation-driven development CLI framework (Node.js, zero runtime dependencies)
**Researched:** 2026-03-28
**Confidence:** HIGH

## Executive Summary

GSD Code-First is a fork of the get-shit-done-cc CLI framework that adds a "code-first" development workflow. Instead of writing specifications before code, developers prototype directly, embed structured `@gsd-tags` in code comments (the ARC annotation system), and extract those annotations as planning input for subsequent phases. The fork extends the existing agentic orchestration infrastructure with a tag scanner, three new agents, five new commands, and modifications to two existing agents — all without touching the upstream command files or introducing any new runtime npm dependencies.

The recommended approach is strictly additive: every new capability ships as a net-new file. The two upstream agents that need new behavior (`gsd-executor` and `gsd-planner`) should be wrapped or extended via config flags, not patched directly — because every patch to an upstream-owned file becomes a permanent merge conflict site that eventually makes upstream sync impossible. The build order is dependency-driven: ARC tag standard first (everything depends on it being stable), then the tag scanner, then the CODE-INVENTORY.md format, then agents, then command orchestration, then installer.

The key risks are fork divergence from careless upstream file modifications, regex false positives polluting the CODE-INVENTORY.md output, and annotation standard churn forcing developers to re-annotate codebases. All three are front-loaded risks that must be resolved in Phase 1. The fourth serious risk — developer adoption failure because annotations feel like overhead — is mitigated structurally by building `gsd-prototyper` early: the prototyper adds ARC tags automatically as it generates code, making the zero-friction path the annotated path.

## Key Findings

### Recommended Stack

The existing codebase is pure Node.js CommonJS with zero runtime npm dependencies, and every addition must maintain this constraint. The tag scanner uses native `RegExp` with the `s` (dotAll) flag and `fs.readdirSync` for recursive file discovery — no external parsing library is needed. All new agent definitions are Markdown files with YAML frontmatter, identical in structure to the 19 existing agents. SDK-layer work (programmatic agent invocation for the `iterate` command) stays in the existing `sdk/` TypeScript subdirectory and uses the already-installed `@anthropic-ai/claude-agent-sdk ^0.2.86`. Tests for CJS code use `node:test` and `assert` (same as all 47 existing test files); SDK tests use vitest. No new devDependencies are needed.

**Core technologies:**
- `node:fs` + `RegExp` (dotAll): tag scanner — language-agnostic extraction from any comment format, zero deps
- `@anthropic-ai/claude-agent-sdk ^0.2.86`: programmatic agent invocation — already in `sdk/package.json`, no version bump needed
- `node:test` + `c8`: CJS test suite — matches all 47 existing test files, enforces 70% line coverage
- Claude Code agent frontmatter (YAML): new agent definitions — exact same format as 19 existing agents, no alternative
- TypeScript ^5.7.0 (SDK layer only): type safety for SDK commands — already in use, boundary must not blur into CJS layer

### Expected Features

The feature set is evaluated against spec-driven tools (Kiro, BMAD-METHOD, OpenSpec) and annotation extractors (leasot, fixme, todos, tickgit). The fork must match the baseline of annotation extractors while delivering the end-to-end planning workflow that none of them provide.

**Must have (table stakes):**
- ARC annotation standard (8-tag vocabulary: `@gsd-context`, `@gsd-decision`, `@gsd-todo`, `@gsd-constraint`, `@gsd-pattern`, `@gsd-ref`, `@gsd-risk`, `@gsd-api`) — without a stable vocabulary, all downstream tooling is meaningless
- Tag scanner with JSON and Markdown output — peer tools (leasot, fixme) provide this as their entire value; this is the floor
- `CODE-INVENTORY.md` artifact — users expect extraction to produce a readable, organized document
- `prototype` command — entry point to code-first mode; without it users have no starting point
- `iterate` command — extract → plan → approve → execute loop; this is the core workflow
- Human approval gate before execution — all trusted agentic tools (Kiro, Codex CLI, OpenSpec) require this
- `annotate` command — most users have existing code; retroactive annotation is essential for adoption
- Preserved original GSD commands — non-negotiable compatibility constraint

**Should have (competitive):**
- Bidirectional code-plan traceability via `@gsd-ref` cross-linking — no peer CLI tool provides this end-to-end
- ARC comment obligation in `gsd-executor` — creates self-documenting codebases automatically during execution
- Code-based planning mode in `gsd-planner` — reads CODE-INVENTORY.md as planning input, not just requirements docs
- `deep-plan` escape hatch — explicit mode-switching that no peer tool offers
- Language-agnostic extraction — competitive advantage over language-specific tools
- `set-mode` per-phase configuration — flexibility that Kiro (one mode) and BMAD (always doc-first) lack

**Defer (v2+):**
- AST-based parsing for any language — complexity with no benefit over regex for structured comments
- IDE / editor integrations — scope creep, mismatches the CLI-first audience
- Multi-repo / monorepo orchestration — separate domain problem
- Real-time watch mode — not expected by CLI users; on-demand extraction is the right model
- Team collaboration / shared annotation server — single-developer tool in v1; git is the collaboration layer

### Architecture Approach

The existing GSD architecture follows four strict patterns that all new code must respect: commands are thin descriptor files with no logic; all logic lives in workflow files or `gsd-tools.cjs`; all reads/writes to `.planning/` go through `gsd-tools.cjs` as a state gateway (never direct `fs.writeFileSync` calls in agent prompts); and agents are spawned by typed names that must match their file names exactly. The Code-First additions insert a new layer between user source code and the existing planning state layer: the ARC annotation layer (tags in code comments) feeds a tag scanner (new `gsd-tools.cjs` subcommands), which produces `CODE-INVENTORY.md` (a new `.planning/` artifact), which is consumed by three new agents and a modified version of `gsd-planner`.

**Major components:**
1. ARC annotation layer — structured `@gsd-tags` embedded in code comments by developers and agents
2. Tag scanner (`lib/tag-scanner.cjs` + `gsd-tools.cjs` subcommands `scan-tags`, `extract-plan`) — regex-based extraction producing JSON and `CODE-INVENTORY.md`
3. `CODE-INVENTORY.md` — new planning artifact in `.planning/`; schema must be locked before either the scanner or the planner agent is written
4. `gsd-prototyper` agent — generates prototype code with ARC annotations baked in
5. `gsd-code-planner` agent — reads CODE-INVENTORY.md and source files, produces PLAN.md
6. `gsd-annotator` agent — retroactively adds ARC tags to existing code
7. New command layer — `/gsd:prototype`, `/gsd:annotate`, `/gsd:extract-plan`, `/gsd:iterate`, `/gsd:deep-plan`, `/gsd:set-mode`
8. Config schema extension — `phase_modes`, `arc`, `default_phase_mode` keys in `lib/config.cjs`

### Critical Pitfalls

1. **Fork divergence via upstream file modification** — Every edit to an upstream-owned file (`gsd-executor.md`, `gsd-planner.md`, `gsd-tools.cjs`, `install.js`) creates a permanent merge conflict site. Implement the ARC comment obligation in a new `gsd-arc-executor.md` wrapper rather than patching `gsd-executor.md`. Maintain a file-level "touched by fork" inventory.

2. **Regex false positives from string/template literal context** — The pattern `@gsd-tag:` appears in string literals, URLs, and template strings, not only in comments. Anchor the regex to comment-opening tokens: `/^\s*(?:\/\/|\/\*|\*|#)\s*@gsd-(\w+):/m`. Build a test fixture file with deliberate edge cases at the same time as the scanner.

3. **ARC tag standard churn after real-world use** — Renaming tags or adding required fields after developers have annotated codebases forces costly re-annotation and destroys trust. Treat the standard as versioned from day one: existing tag names and required fields cannot change, only new optional fields may be added. Run at least one real annotation session before freezing the spec.

4. **Agent prompt drift** — The `gsd-prototyper` and `gsd-code-planner` agents will silently diverge if `CODE-INVENTORY.md` schema changes without updating both. Define the schema as a formal contract with a version field before writing either component.

5. **Install collision with upstream GSD** — Both installers write to `~/.claude/`. Use namespace-scoped marker constants (`GSD_CF_CODEX_MARKER`) and document that this fork replaces (not coexists with) the upstream installation.

## Implications for Roadmap

Based on research, the build order is strictly dependency-driven. Foundation work must be complete before any agent or command is written, because every downstream component depends on the ARC standard being stable and the tag scanner producing correctly structured output.

### Phase 1: Foundation — ARC Standard, Scanner, and Safe Fork Infrastructure

**Rationale:** All 17 downstream components depend on the ARC tag vocabulary being stable and the regex scanner producing correct output. Installing a "no upstream file modification" policy must happen before any code is written, not after. These are the highest-leverage, lowest-complexity tasks in the project, and getting them wrong forces rewrites.

**Delivers:**
- ARC annotation standard document (8-tag vocabulary, syntax rules, versioning policy)
- Config schema extension (`phase_modes`, `arc`, `default_phase_mode` in `lib/config.cjs`)
- `lib/tag-scanner.cjs` with `scan-tags` and `extract-plan` gsd-tools subcommands
- `.gsdignore` support and default exclusion list (node_modules, dist, build, .planning)
- Test fixture file covering all edge cases (string literals, URLs, JSDoc, multiline block comments)
- Namespace-scoped installer markers (`GSD_CF_CODEX_MARKER`)
- File-level inventory: "touched by fork" vs "upstream-owned"

**Addresses features:** ARC annotation standard, tag scanner, per-phase mode configuration
**Avoids pitfalls:** Fork divergence (Pitfall 1), regex false positives (Pitfall 2), standard churn (Pitfall 3), install collision (Pitfall 5), generated code double-counting (Pitfall 11), npx caching (Pitfall 10)

---

### Phase 2: Planning Artifact and Retroactive Annotation

**Rationale:** `CODE-INVENTORY.md` schema must be locked before either the tag scanner or the `gsd-code-planner` agent is finalized, because it is the contract between those two components. The `gsd-annotator` agent enables brownfield adoption and does not depend on `CODE-INVENTORY.md` as input — it can be built in parallel with the inventory format work. The `extract-plan` command is thin and depends only on Phase 1 components.

**Delivers:**
- `CODE-INVENTORY.md` schema with version field (sections, link format, tag grouping, soft size limits)
- `extract-plan` command (`commands/gsd/extract-plan.md`) with `--phase`/`--scope` filter and `--stats` output
- `gsd-annotator` agent (`agents/gsd-annotator.md`) — retroactively adds ARC tags to existing code
- `annotate` command (`commands/gsd/annotate.md`) — spawns annotator, then runs extract-plan

**Addresses features:** CODE-INVENTORY.md artifact, annotate command, brownfield adoption
**Avoids pitfalls:** Agent prompt drift (Pitfall 4), context window saturation (Pitfall 7), adoption failure (Pitfall 8)

---

### Phase 3: Core Agent Layer — Prototyper and Code Planner

**Rationale:** `gsd-prototyper` should be built before the full `iterate` pipeline because it is the primary forcing function for organic annotation: when developers use the prototyper, annotations are added automatically, making the annotated path the zero-effort path. `gsd-code-planner` is the most complex new agent (HIGH risk per architecture research) and requires the CODE-INVENTORY.md schema from Phase 2 to be stable before its prompt is written.

**Delivers:**
- `gsd-prototyper` agent (`agents/gsd-prototyper.md`) — generates prototype code with ARC tags baked in
- `prototype` command (`commands/gsd/prototype.md`) — entry point to code-first mode
- `gsd-code-planner` agent (`agents/gsd-code-planner.md`) — reads CODE-INVENTORY.md + source, produces PLAN.md
- `init iterate` compound context command in `lib/init.cjs` (bundles CODE-INVENTORY.md, config, phase state)
- Modified `gsd-planner` behavior — code-based planning mode gated by `phase_modes` config (additive only)
- Modified `gsd-executor` behavior — implemented as new `gsd-arc-executor.md` wrapper, not as patch to upstream file

**Addresses features:** prototype command, code-based planning mode, ARC comment obligation in executor
**Avoids pitfalls:** Adoption failure (Pitfall 8), fork divergence (Pitfall 1), agent prompt drift (Pitfall 4)

---

### Phase 4: Full Workflow Orchestration — iterate, deep-plan, set-mode

**Rationale:** The `iterate` command is the most complex orchestration piece (depends on all Phase 3 agents). It belongs last in the functional build order. `deep-plan` and `set-mode` are low-complexity commands that depend only on existing upstream commands and the Phase 1 config schema extension. The approval gate and headless/CI patterns must be implemented from the start, following the upstream headless prompt overhaul pattern already established in the codebase.

**Delivers:**
- `iterate` command (`commands/gsd/iterate.md`) — extract-tags → code-planner → approval gate → executor pipeline
- `--non-interactive` flag on `iterate` for CI/scripted use
- `deep-plan` command (`commands/gsd/deep-plan.md`) — wraps existing discuss-phase + plan-phase
- `set-mode` command (`commands/gsd/set-mode.md`) — writes phase_modes config
- Active mode display at command startup for `iterate` and `prototype`
- Updated `gsd:help` listing all new commands

**Addresses features:** iterate command, human approval gate, deep-plan escape hatch, set-mode
**Avoids pitfalls:** Approval blocking in CI (Pitfall 6), invisible mode state (Pitfall 9)

---

### Phase 5: Distribution, Installer, and Documentation

**Rationale:** Installer changes should be made last, after all agent and command files are finalized. Updating the installer before files are stable wastes effort and risks shipping an installer that references files that don't exist yet.

**Delivers:**
- Extended `bin/install.js` — copies new agents and commands to `~/.claude/` with `GSD_CF_` namespaced markers
- Detection and warning when upstream GSD installation is detected
- Updated `package.json` (name, version, `files` array if new directories added)
- Updated README with new commands, workflow diagrams, annotation examples, `.gsdignore` documentation

**Addresses features:** Preserved npm distribution, help documentation, updated installer
**Avoids pitfalls:** Install collision (Pitfall 5), npx caching (Pitfall 10)

---

### Phase Ordering Rationale

- The ARC standard is a blocking dependency for every single downstream component. It must be stable before code is written, not after. This is why Phase 1 is entirely specification and infrastructure work.
- `CODE-INVENTORY.md` schema is a contract between the scanner and the planner agent. Locking it in Phase 2 before either component is finalized prevents the most insidious failure mode (agent prompt drift, Pitfall 4).
- `gsd-prototyper` comes before `gsd-code-planner` because the prototyper drives adoption of organic annotation. If the annotator (`annotate` command) is built first and the prototyper is built later, the natural pattern becomes retroactive annotation — which produces lower-quality planning data and defeats the purpose of code-first.
- `iterate` is built last among functional commands because it depends on all three new agents, the approval gate pattern, and the `init iterate` compound context command. Building it earlier would require constant rework as its dependencies are completed.
- The installer is always last. It copies files. Those files must exist and be stable first.

### Research Flags

Phases likely needing `/gsd:research-phase` during planning:
- **Phase 3 (gsd-code-planner agent):** This is rated HIGH risk in architecture research. The agent must read CODE-INVENTORY.md and produce correctly structured PLAN.md. The prompt engineering for this agent — specifically how it handles large inventories, how it maps tag clusters to work units, and how it handles missing or conflicting tags — is not fully resolved by this research.
- **Phase 4 (iterate command approval gate):** The upstream headless prompt overhaul is the reference implementation, but the specific integration pattern for the `--non-interactive` flag and state cleanup on process kill may need targeted research into the current SDK lifecycle patterns.

Phases with standard patterns (skip research-phase):
- **Phase 1:** Regex patterns, config schema extension, and installer namespace changes are all well-documented patterns. The codebase itself provides all the necessary examples.
- **Phase 2:** `extract-plan` command is thin and mechanical. `gsd-annotator` follows the same agent prompt structure as existing agents.
- **Phase 5:** Distribution, `package.json`, and README follow fully established patterns in the existing codebase.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All technology choices verified against the live codebase: 17 lib files confirmed zero external requires, 47 test files confirmed use node:test, claude-agent-sdk version confirmed via npm view, esbuild version confirmed via npm view |
| Features | HIGH | Feature decisions derived from direct comparison with peer tools (Kiro, BMAD, leasot, tickgit) and from PROJECT.md decisions already recorded in the codebase. The table stakes and anti-features sections are well-grounded. |
| Architecture | HIGH | Architecture research is based on direct inspection of the upstream codebase (workflows, agents, lib files). The patterns described (command-as-descriptor, state gateway, typed subagents) are verified patterns, not inferences. |
| Pitfalls | HIGH (fork/regex) / MEDIUM (adoption) | Fork divergence and regex false positive pitfalls are based on direct code inspection and well-documented regex edge cases. Adoption failure (Pitfall 8) is based on community research patterns — medium confidence because real adoption behavior is inherently unpredictable. |

**Overall confidence:** HIGH

### Gaps to Address

- **gsd-code-planner prompt quality:** The research identifies what inputs and outputs this agent needs, but the specific prompt structure that produces reliable PLAN.md output from CODE-INVENTORY.md is not resolved. This needs a design spike in Phase 3.
- **Context window budget for CODE-INVENTORY.md:** Research recommends a soft size limit but does not quantify it. During Phase 2, measure actual token counts from a representative annotated codebase and set the limit based on data, not guesswork.
- **`gsd-arc-executor.md` implementation pattern:** Research recommends wrapping the upstream executor rather than patching it, but the exact mechanism (prompt injection, `@`-reference chaining, or parallel agent call) needs to be resolved during Phase 3 execution. Check how existing agent composition is done in the codebase.
- **Headless iterate integration:** The upstream headless prompt overhaul is documented in git history but not fully audited against the current `iterate` command requirements. A targeted read of the SDK lifecycle documentation is recommended before implementing the approval gate.

## Sources

### Primary (HIGH confidence)
- Codebase direct audit: `get-shit-done/bin/lib/*.cjs` (17 files), `tests/*.test.cjs` (47 files), `agents/*.md` (19 agents), `sdk/package.json`, `package.json` — core architecture, zero-dep constraint, test patterns
- `npm view @anthropic-ai/claude-agent-sdk version` → `0.2.86` (2026-03-28)
- `npm view esbuild version` → `0.27.4` (2026-03-28)
- `PROJECT.md` recorded decisions — regex over AST rationale, ARC tag vocabulary

### Secondary (MEDIUM confidence)
- [Kiro Specs Documentation](https://kiro.dev/docs/specs/) — spec-driven feature baseline comparison
- [BMAD-METHOD GitHub](https://github.com/bmad-code-org/BMAD-METHOD) — document-first workflow comparison
- [leasot](https://github.com/pgilad/leasot), [fixme](https://github.com/JohnPostlethwait/fixme), [todos](https://github.com/ianlewis/todos) — annotation extractor feature baseline
- [OpenSpec](https://arxiv.org/abs/2602.00180) — approval gate pattern validation
- [Prompt Drift: The Hidden Failure Mode Undermining Agentic Systems](https://www.comet.com/site/blog/prompt-drift/) — Pitfall 4 basis
- [Platform Engineering Maintenance Pitfalls (CNCF 2026)](https://www.cncf.io/blog/2026/01/21/platform-engineering-maintenance-pitfalls-and-smart-strategies-to-stay-ahead/) — fork divergence patterns
- [openedx/code-annotations](https://github.com/openedx/code-annotations) — regex-over-AST rationale for annotation extraction
- [Azure Architecture Center — AI Agent Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) — orchestrator-subagent pattern validation

### Tertiary (LOW confidence)
- [5 Case Studies on Developer Tool Adoption](https://business.daily.dev/resources/5-case-studies-on-developer-tool-adoption/) — Pitfall 8 (adoption failure) behavioral patterns; single source, needs validation against real user feedback
- [Finding Comments in Source Code Using Regular Expressions](https://blog.ostermiller.org/finding-comments-in-source-code-using-regular-expressions/) — regex comment parsing edge cases; older resource, verify against Node.js regex behavior

---
*Research completed: 2026-03-28*
*Ready for roadmap: yes*
