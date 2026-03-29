# Project Research Summary

**Project:** GSD Code-First Fork — v1.1 Autonomous Prototype & Review Loop
**Domain:** CLI framework fork with AI agent orchestration, code annotation parsing, and autonomous coding pipeline
**Researched:** 2026-03-29
**Confidence:** HIGH

## Executive Summary

GSD Code-First v1.1 extends a stable v1.0 foundation by closing the feedback loop between PRD input and working, tested, reviewed code. The four target features — PRD-to-Prototype pipeline, ARC-as-default, Test-Agent (gsd-tester), and Review-Agent (gsd-reviewer) — form a sequential pipeline where each feature is a prerequisite for the next. Competitors like Ralph, quantum-loop, and Addy Osmani's workflow define table-stakes expectations: PRD file input, test execution before review, human approval gates, and actionable review output. GSD differentiates on one key architectural bet: state lives as ARC tags embedded directly in code rather than in external JSON state files, eliminating the sync problems that plague all competitors.

The recommended implementation approach is strictly additive — zero new npm dependencies, two new agent files, two modified commands, one config default change, and one new CJS detection module. The existing orchestrator-subagent pattern (command descriptor spawns agent via Task()) handles all four features without architectural changes. PRD parsing belongs in the command orchestrator (not the agent), test execution belongs in the command layer (not inside gsd-tester), and gsd-reviewer is independent of gsd-verifier despite surface-level similarity.

The critical risks are: autonomous loop divergence (prototype loop must have a hard 3-iteration cap with explicit done-signal checklist), test agent producing vacuous tests (RED-GREEN discipline against @gsd-api contracts rather than stub return values), review verbosity and low-signal output (two-stage Reviewer + Judge pattern, max 5 actionable items), and the ARC-as-default silent breakage for upgrading users (new installs get `true`, existing installs preserve their setting with a one-time migration notice). Pitfall 15 — the `/gsd:review` name collision — is the highest-risk implementation decision and must be resolved before a single line of the review command is written.

---

## Key Findings

### Recommended Stack

The v1.0 zero-runtime-dependency constraint is confirmed and extended to all v1.1 additions. Every new capability uses Node.js built-ins and agent Markdown files only. The only version bump is `@anthropic-ai/claude-agent-sdk` from `^0.2.84` to `^0.2.87` (SDK layer only, no breaking changes in the 0.2.x range). Test framework detection uses a new CJS module (`test-detector.cjs`) that reads the target project's `package.json` with `fs.readFileSync` — no parser library needed.

**Core technologies:**
- Node.js built-ins (CJS layer): All I/O, path resolution, regex — zero runtime dep constraint enforced; no additions needed for v1.1
- `@anthropic-ai/claude-agent-sdk@^0.2.87`: Programmatic agent invocation — SDK layer only; bump from `^0.2.84`, no breaking changes confirmed
- `child_process.execSync`: Test execution in review command orchestrator — already used in gsd-tools.cjs, no new pattern
- Agent Markdown files (YAML frontmatter): gsd-tester and gsd-reviewer follow the exact format of all 23 existing agents
- `test-detector.cjs` (new): Detects vitest/jest/mocha/ava/node:test from target project's `package.json`; covers >95% of Node.js projects

**New files required (v1.1):**
- `agents/gsd-tester.md` — test generation agent
- `agents/gsd-reviewer.md` — review and evaluation agent
- `get-shit-done/bin/lib/test-detector.cjs` — test framework detection module
- `commands/gsd/prototype.md` — modified to add `--prd <path>` flag
- `commands/gsd/review-code.md` (new, distinct from existing `review.md`) — see Pitfall 15

### Expected Features

**Must have (table stakes):**
- PRD file as structured input to `/gsd:prototype` — every autonomous coding pipeline in 2026 reads a spec/PRD file; without it the prototype has no traceable connection to requirements
- Acceptance criteria from PRD become `@gsd-todo` tags embedded in code — makes each AC directly executable via `/gsd:iterate`
- Test agent writes runnable tests, not stubs — must execute and confirm green; stub-only output is a documented adoption killer
- Code correctness evaluation when `/gsd:review` (or `/gsd:review-code`) is called after a prototype — users do not expect plan review at that workflow stage
- Review output includes actionable next steps with file path, severity, and a concrete action — findings dump without recommended actions is universally cited as a failure mode
- Human approval gate preserved across all flows — `--non-interactive` as CI escape hatch only
- ARC annotations always enabled by default — opt-in silently omits the core feature for new users

**Should have (competitive differentiators):**
- PRD ACs become machine-readable `@gsd-todo` tags (no competitor closes this loop end-to-end)
- Test-Agent annotates untested paths with `@gsd-risk` tags — risk is visible in planning artifacts, not silently ignored
- Two-stage review: spec compliance (Stage 1) before code quality (Stage 2) — validates requirements before wasting cycles on style or security
- Structured `REVIEW-CODE.md` output schema designed for future `--fix` chaining — design the schema now; retroactive restructuring is expensive
- ARC always-on simplifies `/gsd:iterate` step 4 routing (removes the `config-get arc.enabled` conditional branch)

**Defer to v1.1+ or v2:**
- Review-to-iterate `--fix` flag (pipe REVIEW-CODE.md findings as `@gsd-todo` tags into iterate)
- PRD template scaffolding (`/gsd:prototype --init-prd`)
- `@gsd-coverage` tag type for test coverage gaps in CODE-INVENTORY.md
- Parallel test generation (coordination complexity exceeds benefit for single-developer workflow)
- Multi-language test runner detection beyond Node.js

### Architecture Approach

The v1.1 system extends the existing five-layer architecture (entry points, workflow layer, agent layer, CLI tools layer, planning state layer) with two new agents and modifications to two commands. All changes are strictly additive — no upstream file is modified. The key architectural rule: PRD ingestion belongs in the `/gsd:prototype` command orchestrator, not in the gsd-prototyper agent; test execution belongs in the `/gsd:review-code` command layer, not inside gsd-tester. These boundaries prevent agents from coupling to specific input formats and prevent context window blockage on long-running test suites.

**Major components:**
1. `/gsd:prototype` (modified) — ingests PRD via `--prd <path>`, enriches Task() prompt sent to gsd-prototyper, auto-runs extract-plan; gsd-prototyper agent itself is unchanged
2. `gsd-tester` (new agent) — writes unit/integration tests against @gsd-api contracts using RED-GREEN discipline; scoped to PROTOTYPE-LOG.md file list to bound cost on large codebases
3. `/gsd:review-code` (new command, distinct from existing `/gsd:review`) — orchestrates: spawn gsd-tester, run tests via Bash, capture results, spawn gsd-reviewer with results as context
4. `gsd-reviewer` (new agent) — two-stage evaluation (spec compliance then code quality via Reviewer + Judge pattern), produces `REVIEW-CODE.md` with max 5 prioritized next steps
5. `lib/config.cjs` (modified) — `arc.enabled` defaults to `true` for new installs only via `config.arc?.enabled ?? true`; existing explicit `false` configs preserved
6. `test-detector.cjs` (new CJS module) — detects test framework from target project's package.json

**Build order (dependency-driven):**
- Layer 1 (no dependencies): `arc.enabled` config default change
- Layer 2 (no dependencies): `/gsd:prototype` PRD flag
- Layer 3 (no dependencies): `gsd-tester` agent + `test-detector.cjs`
- Layer 4 (after Layer 3 output format is settled): `gsd-reviewer` agent
- Layer 5 (after Layers 3-4): `/gsd:review-code` command
- Layer 6 (after Layers 3-4): `bin/install.js` additions

Layers 2 and 3 can be built in parallel. Layer 5 is the integration point and must be built last among code changes.

### Critical Pitfalls

1. **Autonomous loop divergence (Pitfall 12)** — Hard cap prototype iterations at 3; emit an explicit done-signal checklist from PRD acceptance criteria before the loop starts; track attempts in a structured `## Attempts` section of PROTOTYPE-LOG.md; when cap is hit, surface what was produced rather than failing silently. Never combine "prototype" (one-shot scaffold) with "iterate" (approval-gated refinement) in the same autonomous loop.

2. **Vacuous test generation (Pitfall 13)** — RED-GREEN discipline is mandatory: tests must fail against scaffold stubs, pass only after real implementation; write tests against @gsd-api contracts, not against stub return values; include a test audit sub-pass that flags any assertion that would pass on `return null`. Never let "tests pass" be the sole success signal.

3. **Review verbosity and low-signal output (Pitfall 14)** — Implement two-stage Reviewer + Judge pattern (HubSpot production-validated); cap output at 5 prioritized items per review; each item must include file path, line range, severity (HIGH/MEDIUM/LOW), and a concrete action ("change X to Y"), not a vague suggestion; distinguish test-coverage review, implementation review, and ARC annotation review as separate focused passes.

4. **`/gsd:review` name collision (Pitfall 15)** — Name the new command distinctly at `commands/gsd/review-code.md`; never write it at `commands/gsd/review.md`; add explicit disambiguation notes to both command files; this decision must be made and locked before Phase 4 begins.

5. **ARC-as-default silent breakage for upgraders (Pitfall 17)** — New installs get `arc.enabled: true`; existing installs with explicit `arc.enabled: false` are left unchanged; print a one-time migration notice on upgrade; `iterate` must always log which executor it selected, making the ARC/non-ARC distinction visible at runtime.

---

## Implications for Roadmap

Based on combined research, the phase structure is dictated by hard feature dependencies: the ARC config default affects all downstream behavior and costs almost nothing to land first; the prototype pipeline and test agent are independent and can be built in parallel; the review agent conceptually depends on the test agent's output format being settled; the review command is the integration point and must be built last.

### Phase 1: ARC-as-Default + Config Foundation

**Rationale:** Zero-risk config change with no dependencies. Landing this first means all subsequent development and testing reflects the intended default. Enables downstream simplification of `/gsd:iterate` step 4 routing. Costs one line in `lib/config.cjs`; the only real work is the upgrade detection and migration notice.
**Delivers:** `arc.enabled` defaults to `true` for new installs; upgrade migration notice for existing `false` configs; `iterate` logs which executor it selected at runtime.
**Addresses:** ARC-as-default table stakes; simplifies executor routing.
**Avoids:** Pitfall 17 (silent breakage on upgrade) by scoping the default change to new installs only and surfacing a migration notice.

### Phase 2: PRD-to-Prototype Pipeline

**Rationale:** Independent of the test and review agents. Can be built and validated immediately after Phase 1. Delivers the most visible new entry point for the tool. The PRD parsing approach — semantic LLM extraction by the command orchestrator, not structural regex — handles free-form PRDs without a parser library and avoids Pitfall 16 provided a requirements-found confirmation step is included before scaffold generation begins.
**Delivers:** `--prd <path>` flag on `/gsd:prototype`; PRD context injected into gsd-prototyper Task() prompt; requirements-found confirmation step before scaffold generation; graceful missing-PRD handling with scaffold offer.
**Addresses:** PRD-as-structured-input table stakes; ACs-become-@gsd-todo differentiator.
**Avoids:** Pitfall 16 (PRD parsing brittleness) via semantic extraction with explicit requirements confirmation; Pitfall 12 (loop divergence) by keeping the prototype as a single-shot scaffold, not an autonomous loop.
**Uses:** No new dependencies; command orchestrator pattern identical to existing `/gsd:iterate`.

### Phase 3: gsd-tester Agent + test-detector.cjs

**Rationale:** Independent of Phase 2; can be built in parallel. Produces the test suite that the review agent in Phase 4 depends on. The `test-detector.cjs` CJS module is a prerequisite for gsd-tester knowing which test framework to target. Building gsd-tester before gsd-reviewer allows the reviewer's input format to be defined against real gsd-tester output, not assumptions.
**Delivers:** `test-detector.cjs` module covering vitest/jest/mocha/ava/node:test; `gsd-tester.md` agent writing tests against @gsd-api contracts with RED-GREEN discipline; @gsd-risk annotations on untested paths; test files scoped to PROTOTYPE-LOG.md file list.
**Addresses:** Test-agent-writes-runnable-tests table stakes; @gsd-risk-from-test-agent differentiator.
**Avoids:** Pitfall 13 (vacuous tests) via RED-GREEN enforcement and contract-based assertions; Pitfall 7 (context window saturation) by scoping to PROTOTYPE-LOG.md file list.
**Implements:** gsd-tester architecture component; test-detector.cjs CJS module.

### Phase 4: gsd-reviewer Agent + /gsd:review-code Command

**Rationale:** Integration point — depends on Phase 3 output format being settled. The command name (`/gsd:review-code`, not `/gsd:review`) must be locked before writing any code. The two-stage Reviewer + Judge architecture must be designed in the agent prompt before the command orchestrates it. The Judge prompt — operationalizing "succinct, accurate, actionable" — is the highest-complexity design task in this phase.
**Delivers:** `gsd-reviewer.md` agent with two-stage evaluation; `/gsd:review-code` command at `commands/gsd/review-code.md`; `REVIEW-CODE.md` output file (distinct from existing `REVIEWS.md`); max 5 prioritized next steps each with file path, severity, and concrete action; disambiguation notes in both review command files.
**Addresses:** Review-evaluates-code table stakes; actionable-next-steps table stakes; two-stage review differentiator; structured output schema for future `--fix` chaining.
**Avoids:** Pitfall 14 (verbose low-signal output) via two-stage Reviewer + Judge pattern and output cap; Pitfall 15 (command name collision) via distinct command name and file path; Pitfall 4 (agent prompt drift) by defining REVIEW-CODE.md schema as an explicit contract.

### Phase 5: Installer + Integration Validation

**Rationale:** Mechanical last step — register new agent files in `bin/install.js`, run an end-to-end pipeline test (PRD in, prototype out, tests written, review produced), verify all artifacts land correctly, confirm no upstream file collisions.
**Delivers:** `gsd-tester.md` and `gsd-reviewer.md` registered for copy to `~/.claude/agents/`; `test-detector.cjs` registered; end-to-end smoke test; updated help output listing new commands.
**Addresses:** Installation compatibility.
**Avoids:** Pitfall 1 (fork divergence) by verifying `git diff upstream/main` shows no modifications to upstream-owned files; Pitfall 5 (install collision) via `GSD_CF_`-namespaced markers.

### Phase Ordering Rationale

- Phase 1 first because the ARC default affects all downstream behavior; it is also the lowest-risk change and confirms the config upgrade detection path before any new feature depends on it.
- Phases 2 and 3 can be built in parallel; they share no dependencies. Sequencing them depends on team capacity or developer preference.
- Phase 4 must follow Phase 3 because gsd-reviewer depends on gsd-tester's output format being settled. The reviewer must know what test execution output looks like before its prompt is written.
- Phase 5 is always last. Files must be stable and tested before the installer copies them.
- The command naming decision (Pitfall 15: use `/gsd:review-code`) is a blocking decision for Phase 4. It must be locked before Phase 4 begins, not treated as an implementation detail.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 2 (PRD pipeline):** The requirements extraction approach needs a concrete agent prompt validated against at least 2-3 real PRD formats (canonical template, prose-only, table-based spec) before the command is written. The requirements-found confirmation UX — how the user approves or corrects the parsed list — needs a concrete design.
- **Phase 3 (gsd-tester):** RED-GREEN discipline against scaffold stubs is architecturally sound but requires a concrete definition of "stub implementation" that the gsd-tester agent can detect. This definition must be in the agent prompt before it is written, not discovered during testing.
- **Phase 4 (review command and Judge agent):** The two-stage Reviewer + Judge pattern needs an explicit Judge prompt that operationalizes "succinct, accurate, actionable" in terms the LLM can execute consistently. This prompt design is the hardest engineering problem in v1.1.

Phases with standard patterns (skip research-phase):
- **Phase 1 (ARC config default):** One-line config change with a well-understood backward-compatibility pattern already used elsewhere in `config.cjs`. No research needed.
- **Phase 5 (installer):** Mechanical file registration following the established pattern in `bin/install.js`. No research needed.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All decisions verified against the live codebase; zero-dep constraint confirmed by reading all 17 lib files; SDK version verified via npm view on 2026-03-29; no new runtime dependencies required |
| Features | HIGH | Table stakes validated across quantum-loop (WebFetch confirmed), Ralph, and Addy Osmani (WebFetch confirmed); anti-features backed by TiCODER research and Codepipes anti-patterns; existing command files read directly |
| Architecture | HIGH | Based on direct inspection of 23 agents, 63 commands, all workflow files, and gsd-tools.cjs; build order derived from actual dependency analysis; anti-patterns identified from direct codebase reading |
| Pitfalls | HIGH (v1.0, fork/config) / MEDIUM (v1.1, autonomous loops, test quality) | v1.0 pitfalls: verified by direct codebase analysis. v1.1 pitfalls 12-17: HIGH on fork compatibility and config changes; MEDIUM on autonomous loop prevention and test generation quality (research-backed via TiCODER and HubSpot case study, but not yet validated in this specific codebase) |

**Overall confidence:** HIGH

### Gaps to Address

- **Review command naming (Pitfall 15 vs Architecture Option A):** Architecture research recommends a context-aware `/gsd:review` command (Option A), while Pitfalls research recommends a distinct `/gsd:review-code` (Option B). This is an unresolved design decision. Recommendation: `/gsd:review-code` (Pitfall 15 makes the collision consequences explicit and severe; Option A's backward-compatible routing adds complexity without eliminating the confusion risk).
- **PRD template design:** The pipeline needs a canonical PRD template for the "full structure" happy path, but must degrade gracefully on unstructured PRDs. The exact template structure is not specified in research. This is a UX design task for Phase 2 planning.
- **REVIEW-CODE.md vs REVIEW.md file naming:** FEATURES.md specifies `REVIEW-CODE.md` as a hard naming constraint to avoid overwriting the existing `REVIEWS.md` plan-review artifact. ARCHITECTURE.md refers to `.planning/prototype/REVIEW.md`. These must be reconciled in Phase 4 planning. Recommendation: `REVIEW-CODE.md` per FEATURES.md's explicit hard-constraint note.
- **gsd-tester scope for brownfield (non-prototype) codebases:** Research confirms the agent can run on existing codebases without a PROTOTYPE-LOG.md, but the scoping mechanism for that path is not defined. Validate and document during Phase 3 planning.
- **Judge agent prompt design:** The Reviewer + Judge two-stage pattern is validated by HubSpot's production experience (Pitfall 14 source), but no concrete Judge prompt exists in the research. This must be designed and validated before Phase 4 implementation begins.

---

## Sources

### Primary (HIGH confidence)
- GSD Code-First codebase (direct read, 2026-03-29): `agents/*.md` (23 agents), `commands/gsd/*.md` (63 commands), `get-shit-done/bin/lib/*.cjs` (17 lib files), `get-shit-done/workflows/*.md` — zero-dep constraint, patterns, build order
- `.planning/PROJECT.md` — v1.1 target features and constraints
- `npm view @anthropic-ai/claude-agent-sdk version` → `0.2.87` (2026-03-29)
- `npm view vitest version` → `4.1.2` (2026-03-29)
- [Agentic Coding Trends Report 2026 — Anthropic](https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf) — table stakes patterns
- [Addy Osmani — LLM coding workflow going into 2026](https://addyosmani.com/blog/ai-coding-workflow/) — approval gate patterns; table stakes validation
- [quantum-loop — spec-driven with DAG execution and two-stage review gates](https://github.com/andyzengmath/quantum-loop) — two-stage review architecture; PRD pipeline comparison
- [Software Testing Anti-Patterns — Codepipes](https://blog.codepipes.com/testing/software-testing-antipatterns.html) — 100% coverage anti-pattern; test generation quality floor

### Secondary (MEDIUM confidence)
- [Ralph Loop — PRD-driven autonomous coding loop](https://github.com/snarktank/ralph) — PRD input format comparison
- [Create custom subagents — Claude Code Docs](https://code.claude.com/docs/en/sub-agents) — agent toolset and Bash execution patterns
- [How we use Claude Agents to automate test coverage — DEV Community](https://dev.to/melnikkk/how-we-use-claude-agents-to-automate-test-coverage-3bfa) — agent-driven test generation pattern
- [Create Reliable Unit Tests with Claude Code — DEV Community](https://dev.to/alfredoperez/create-reliable-unit-tests-with-claude-code-4e8p) — context-management for test agents
- [AI Code Review Tools: 8 Options for the Agent Era — Codegen](https://codegen.com/blog/ai-code-review-tools/) — review output quality patterns; actionable next-steps as table stakes
- [TiCODER research — test-driven interactive code generation (Penn)](https://www.seas.upenn.edu/~asnaik/assets/papers/tse24_ticoder.pdf) — test generation quality findings; vacuous test patterns
- [AI Agent Anti-Patterns Part 1 — Allen Chan, March 2026](https://achan2013.medium.com/ai-agent-anti-patterns-part-1-architectural-pitfalls-that-break-enterprise-agents-before-they-32d211dded43) — agent loop divergence patterns

### Tertiary (LOW confidence)
- [Writing PRDs for AI Code Generation Tools in 2026 — ChatPRD](https://www.chatprd.ai/learn/prd-for-ai-codegen) — PRD-to-code agent patterns (marketing content, used only for pattern validation)
- [The State of AI Coding Agents 2026 — Dave Patten](https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a) — market context for competitive positioning

---

*Research completed: 2026-03-29*
*Ready for roadmap: yes*
