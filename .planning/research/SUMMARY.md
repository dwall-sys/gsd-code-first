# Project Research Summary

**Project:** GSD Code-First Fork — v1.2 (Brainstorm, Feature Map, Architecture Mode)
**Domain:** CLI-based autonomous coding workflow with code-embedded planning state
**Researched:** 2026-03-29
**Confidence:** HIGH

## Executive Summary

GSD Code-First v1.2 adds three tightly coupled capabilities to an already working v1.1 pipeline: a `/gsd:brainstorm` conversational command that produces structured PRDs, a Feature Map auto-aggregator that derives a live `FEATURES.md` from PRDs and `@gsd-tags` in code, and an `--architecture` flag on `/gsd:prototype` that enables skeleton-first scaffolding before implementation begins. All three features extend the existing five-layer architecture (entry-point commands, agent workers, CLI dispatch tools, lib modules, planning artifacts) without introducing new layers or runtime npm dependencies. The zero-dep constraint that has held through v1.0 and v1.1 is fully maintained: all new capabilities are implemented as Markdown command/agent files, one new `feature-aggregator.cjs` lib module using only Node.js built-ins, and two new gsd-tools switch cases.

The recommended approach treats the PRD as the central contract artifact. Brainstorm writes it, prototype consumes it, and the Feature Map derives status from the gap between what the PRD planned and what code tags confirm is implemented. This PRD-as-contract model avoids the most dangerous failure mode in tools of this type: silent scope loss at the brainstorm-to-prototype handoff. All three features chain naturally — brainstorm auto-triggers feature map generation, prototype auto-triggers tag extraction and feature map update — making the pipeline self-maintaining when used as designed. The command-orchestrator / agent-worker split established in v1.0 is applied consistently: all flag parsing, file I/O, and auto-chaining lives in command files; all judgment-dependent reasoning lives in agent files; all deterministic parsing lives in the lib module.

The critical risks cluster around three handoff points: (1) schema drift between brainstorm PRD output and prototype AC extractor expectations — prevented by defining one canonical PRD schema before writing either command; (2) feature map staleness when regeneration is not coupled tightly to tag extraction — prevented by making FEATURES.md regeneration part of every `extract-tags` run; and (3) AC numbering collision across multiple brainstorm sessions — prevented by requiring brainstorm to read existing PRD.md and continue numbering from the current maximum rather than resetting. Architecture Mode introduces a secondary risk: generated skeletons diverging from project conventions. This is prevented by requiring the prototyper to read existing file structure and naming patterns before generating any files.

---

## Key Findings

### Recommended Stack

The v1.2 stack is identical to v1.1 — Node.js built-ins only, no new runtime dependencies. New capabilities are implemented as Markdown command and agent files plus a `feature-aggregator.cjs` lib module using only `fs`, `path`, and `String` built-ins. The module follows the exact same shape as `arc-scanner.cjs`: focused utility, pure functions, covered by `node:test` unit tests. Two new `gsd-tools.cjs` switch cases (`list-prds`, `aggregate-features`) are thin dispatch entries that require the new lib module. No new directories. All new files fit into existing `agents/` and `commands/gsd/` paths already in `package.json`'s `files` array.

**Core technologies:**
- Node.js built-ins (`fs`, `path`, `RegExp`): all I/O and string parsing — zero external dependencies maintained throughout v1.2
- `feature-aggregator.cjs` (new lib module): PRD line-scanning, CODE-INVENTORY.md table parsing, structural merge with contradiction detection — ~150 lines, no external deps
- `gsd-tools.cjs` switch cases `list-prds` and `aggregate-features`: thin CLI dispatch; all logic delegated to feature-aggregator.cjs
- Claude Code YAML agent frontmatter: two new agent files (`gsd-brainstormer.md`, `gsd-feature-mapper.md`) — identical format to all 25 existing agents; auto-copied by installer wildcard scan, no explicit registration
- Markdown command files: one new command (`brainstorm.md`), one modified (`prototype.md` — `--arch` flag only, additive), one possibly modified (`extract-plan.md` — optional feature-map trigger)

**Critical version note:** `@anthropic-ai/claude-agent-sdk` is `^0.2.87` as of 2026-03-29 (bumped from `^0.2.84` in v1.1). No breaking changes in the 0.2.x range; carry forward without version bump needed for v1.2.

**What NOT to add:** `gray-matter`/`front-matter` for YAML parsing (PRD frontmatter keys are single-value strings resolvable with `line.split(':')[1].trim()`); `remark`/`unified` for Markdown AST (Feature Groups sections are fixed-format; line scanning is sufficient); a separate `/gsd:architecture` command (architecture mode is a flag variant, not a new workflow).

### Expected Features

**Must have (table stakes — v1.2 launch):**
- Brainstorm asks targeted, one-at-a-time clarifying questions before generating PRD — every capable AI PRD tool uses conversational back-and-forth; a command that dumps a PRD after one prompt feels unreliable
- Brainstorm output is a valid `.planning/PRD.md` in the exact format `/gsd:prototype` expects — the chain must work without manual reformatting; the two commands form a pipeline
- Architecture Mode (`--arch` flag) produces a real runnable skeleton (directories, config, typed interfaces, `@gsd-context` + `@gsd-decision` tags) — not documentation stubs
- Architecture Mode is a flag on existing `/gsd:prototype`, not a separate command — consistent with how Bolt and v0 extend existing commands with modes
- Feature Map (`FEATURES.md`) is readable as a project status summary — aggregates from PRD ACs and live code tags; answers "what's in this project and where does it stand?"

**Should have (competitive differentiators — include if capacity permits):**
- Brainstorm feature grouping with cross-feature dependency analysis — Kiro and Bolt produce flat lists; GSD groups features and surfaces cross-feature dependencies that directly inform roadmap phase structure
- Feature Map completion status derived from code state — `@gsd-todo` tag presence/absence confirms which ACs are in-progress vs. unstarted; surfaces ambiguity explicitly
- Architecture Mode embeds `@gsd-decision` tags on every structural choice — makes the "why" of the architecture visible to future `/gsd:iterate` runs via CODE-INVENTORY.md; no other tool closes this loop
- Feature Map `## Needs Clarification` section explicitly surfaces contradictions between PRD ACs and code tag state rather than silently picking a status

**Defer (v2+):**
- Multi-PRD phase splitting in brainstorm — requires ROADMAP.md with phase structure to already exist; complex; high value but distinct enough to be its own milestone
- Brainstorm-from-existing-codebase (brownfield reverse PRD) — high value but distinct enough to be its own milestone
- Spec-as-source mode — FEATURES.md becomes source of truth; identified by Martin Fowler's SDD research as the hardest and most fragile approach
- Feature Map with line-level code coverage links — requires full `@gsd-ref` discipline; too fragile to require at v1.2

**Explicit anti-features to avoid:**
- Auto-chaining brainstorm directly into prototype without a human approval stop — removes the mandatory PRD review gate; Kiro's explicit human checkpoint between each phase is the validated pattern
- Making FEATURES.md manually editable — next aggregation run silently overwrites edits; use a `FEATURES-NOTES.md` sidecar if persistent annotations are needed
- Architecture Mode generating full working app skeletons for any stack — delegates to dedicated tools (`create-next-app`, etc.); GSD focuses on structural decisions and annotation

### Architecture Approach

The system extends the existing five-layer architecture with new components that follow established patterns without exception. The Command-Orchestrator / Agent-Worker split is the governing pattern: commands handle all orchestration (flag parsing, file I/O, approval gates, Bash auto-chains), agents perform judgment-dependent synthesis (conversation flow, PRD drafting, semantic feature categorization), and `feature-aggregator.cjs` handles all deterministic operations (AC parsing, table parsing, deduplication, contradiction detection). Architecture Mode is the cleanest example: the `--arch` flag causes the command to inject a different Task() context string into the existing `gsd-prototyper` agent — zero new agent files, zero new mechanisms.

**Major components:**
1. `commands/gsd/brainstorm.md` (new) — orchestrates conversation loop, approval gate, PRD write, auto-chains `aggregate-features`; never writes PRD directly without user confirmation
2. `agents/gsd-brainstormer.md` (new) — drives structured conversation with running decision ledger; returns PRD draft as Task() string; never writes files directly
3. `agents/gsd-feature-mapper.md` (new) — semantic synthesis on top of feature-aggregator output; categorizes features, identifies genuine conflicts, writes "why" prose; handles semantic deduplication the lib module cannot
4. `get-shit-done/bin/lib/feature-aggregator.cjs` (new) — deterministic parsing and merge: AC line extraction from PRD, `@gsd-tag` parsing from CODE-INVENTORY.md, structural merge, contradiction flagging with `[DIVERGED]` markers, format output
5. `gsd-tools.cjs` cases `list-prds` and `aggregate-features` (new) — thin dispatch; all logic in feature-aggregator.cjs following the exact `extract-tags` case pattern
6. `.planning/FEATURES.md` (new artifact) — derived, never edited manually; regenerated on every `extract-tags` run; includes `last-updated` and `source-hash` header

**Files that do NOT need modification:** `agents/gsd-prototyper.md` (arch-mode via Task() context only), `get-shit-done/bin/lib/arc-scanner.cjs` (feature aggregation is separate), `commands/gsd/iterate.md` (FEATURES.md is outside the iterate loop), `bin/install.js` (new agents auto-copied via wildcard scan), all original upstream GSD commands.

**Build sequence (dependency-driven):**
- Phase 1: `feature-aggregator.cjs` + gsd-tools cases (shared infrastructure, build and test first)
- Phase 2a (parallel): `brainstorm.md` + `gsd-brainstormer.md` (independent entry point)
- Phase 2b (parallel): `--arch` flag in `prototype.md` (completely independent, additive only)
- Phase 3: `gsd-feature-mapper.md` + FEATURES.md schema + auto-chain wiring (requires Phase 1 stable + real PRD from Phase 2a)
- Phase 4: Integration validation across full pipeline

### Critical Pitfalls

1. **Brainstorm-to-Prototype PRD Schema Drift (Pitfall 21)** — Brainstorm and prototype developed independently converge on different PRD formats; pipeline loses ACs silently at the handoff. Prevention: define one canonical PRD schema in `get-shit-done/references/prd-format.md` BEFORE writing either command; add a structural validation step in brainstorm that runs the same AC extractor prototype uses and shows the user the AC count before writing PRD.md. This is the highest-risk failure mode in v1.2.

2. **Feature Map Staleness (Pitfall 23)** — FEATURES.md becomes stale the moment any tag is resolved or any PRD section is updated. Prevention: couple FEATURES.md regeneration to every `extract-tags` run — not a standalone on-demand command. Add `last-updated` and `source-hash` to the FEATURES.md header so readers know immediately whether it reflects current code state.

3. **AC Numbering Collision Across Brainstorm Sessions (Pitfall 27)** — A second brainstorm session resets AC numbering to AC-1, colliding with `@gsd-todo(ref:AC-N)` tags already in code from the first session. Prevention: brainstorm reads existing PRD.md before starting and continues AC numbering from the current maximum; prototype validates that AC numbers it is about to embed don't collide with existing tags in CODE-INVENTORY.md.

4. **Architecture Mode Skeleton Diverging from Project Conventions (Pitfall 25)** — Generated skeleton uses agent training-data defaults instead of the project's actual naming, module system, and directory conventions. Prevention: architecture mode must read existing file structure and naming patterns before generating any files; for new projects, asks the user for convention choices before generating; skeleton shown to user for approval before any files are written.

5. **Conversational Brainstorm Losing Structured Data (Pitfall 22)** — Negative decisions (constraints, exclusions, deferred features) are omitted from the PRD during synthesis because LLMs default to forward-looking positive documents. Prevention: gsd-brainstormer maintains a running structured decision ledger throughout the conversation; user approves the ledger before PRD.md is written; final PRD always includes `## Out of Scope` and `## Constraints` sections.

---

## Implications for Roadmap

The three v1.2 features decompose into four implementation phases driven by dependency order. Feature C (Architecture Mode) is fully independent and can be built in parallel with Feature A (Brainstorm). Feature B (Feature Map) depends on shared infrastructure (Phase 1) and benefits from real PRD input (from Phase 2a) to validate merge fidelity.

### Phase 1: Feature Aggregation Infrastructure

**Rationale:** `feature-aggregator.cjs` is shared infrastructure for both the Feature Map command and the brainstorm auto-chain. Building it first and covering it with `node:test` unit tests ensures the foundation is solid before any command wires into it. This is the same build-lib-before-commands pattern used for `arc-scanner.cjs` in v1.0. It is the lowest-risk, highest-leverage starting point.
**Delivers:** `feature-aggregator.cjs` lib module with functions: `readPrdFeatures`, `readCodeFeatures`, `mergeFeatures`, `formatFeatureMap`; `list-prds` gsd-tools case; `aggregate-features` gsd-tools case with `--input`, `--output`, `--mode` flags; unit tests for AC line parsing, CODE-INVENTORY.md table parsing, merge logic, and contradiction detection.
**Addresses:** Feature Map infrastructure (P1 from FEATURES.md prioritization matrix)
**Avoids:** Architecture Anti-Pattern 5 (coupling aggregation into arc-scanner.cjs); Pitfall 24 (contradictory feature status silently resolved — `[DIVERGED]` markers and `## Needs Clarification` section built into aggregator from day one)

### Phase 2a: Brainstorm Command and Agent

**Rationale:** Brainstorm is an entry point with no upstream dependencies — it initializes the PRD that everything else reads. Can be built and validated on a fresh project before Feature Map exists. The PRD schema reference file (`prd-format.md`) is the first deliverable of this phase and must be frozen before either the brainstormer agent or the feature-aggregator's AC parser is finalized.
**Delivers:** Canonical PRD schema in `get-shit-done/references/prd-format.md` (frozen contract); `/gsd:brainstorm` command with `--force`, `--prd-path`, `--non-interactive` flags; `gsd-brainstormer.md` agent with structured decision ledger, max-7-questions constraint, feature grouping, dependency analysis; brainstorm → PRD.md pipeline with user approval gate; PRD.md structural validation showing AC count before write.
**Addresses:** P1: brainstorm command + agent, prototype-compatible PRD output; P2: feature grouping + dependency analysis
**Avoids:** Pitfall 21 (schema drift — canonical schema defined first); Pitfall 22 (lost structured data — decision ledger required); Pitfall 27 (AC numbering collision — read existing PRD before starting); Architecture Anti-Pattern 1 (brainstormer writing PRD directly without approval)

### Phase 2b: Architecture Mode Flag (parallel with 2a)

**Rationale:** Completely independent — only adds `--arch` flag handling to `prototype.md` and injects a different Task() context string into the existing `gsd-prototyper`. No new agent files, no new lib modules, no dependency on phases 1 or 2a. Parallel build is safe and efficient.
**Delivers:** `--architecture` flag on `/gsd:prototype` with convention-reading step before generation; two-pass flow (skeleton → user confirmation → implementation pass) with `--non-interactive` bypass; skeleton output with `@gsd-decision` and `@gsd-context` on every module boundary; `ARCHITECTURE-NOTES.md` listing modules, interface contracts, dependency graph; skeleton audit gate (warn if files > ACs * 2); `arc.architecture_mode` config key in VALID_KEYS.
**Addresses:** P1: `--architecture` flag, runnable skeleton output; P2: `@gsd-decision` annotation discipline
**Avoids:** Pitfall 25 (skeleton diverging from conventions — convention-reading step mandatory); Pitfall 26 (overly deep skeleton — PRD AC count as scope bound, skeleton audit gate); Architecture Anti-Pattern 2 (separate gsd-arch-prototyper.md agent — flag + context injection instead)

### Phase 3: Feature Map Agent, Artifact Schema, and Integration Wiring

**Rationale:** Requires Phase 1 infrastructure to be stable. Requires at least one real PRD from Phase 2a to validate semantic deduplication against realistic input. `gsd-feature-mapper` performs semantic synthesis on top of `feature-aggregator.cjs` structural output — the lib module must be proven before the agent layer is built on top of it.
**Delivers:** `gsd-feature-mapper.md` agent for semantic categorization, conflict identification, and "why" prose; canonical FEATURES.md schema with `last-updated`/`source-hash` header; FEATURES.md regeneration coupled to every `extract-tags` run; auto-chain wired into `brainstorm.md` Step 7 and `prototype.md` Step 5b; `## Needs Clarification` section for contradictions; PRD AC count vs. CODE-INVENTORY.md tag count delta reported explicitly.
**Addresses:** P1: Feature Map command + FEATURES.md output; P2: completion status from code state, contradiction surfacing
**Avoids:** Pitfall 23 (staleness — regeneration coupled to extract-tags, not standalone); Pitfall 24 (contradictory status silently resolved — `## Needs Clarification` section); Architecture Anti-Pattern 3 (running aggregate-features from inside an agent — command layer owns the auto-chain)

### Phase 4: Integration Validation and Help Updates

**Rationale:** After all three features work independently, validate the full chain end-to-end. Confirm no regressions in existing v1.1 commands. Update help output to surface new commands. This is always the final phase — files must be stable before integration is asserted.
**Delivers:** End-to-end smoke test: brainstorm → PRD.md → prototype --arch → CODE-INVENTORY.md → FEATURES.md; `/gsd:help` update listing `/gsd:brainstorm` and `/gsd:feature-map`; validation that all original upstream GSD commands work unchanged; `git diff upstream/main` shows no modifications to upstream-owned files.
**Avoids:** Pitfall 1 (fork divergence — upstream file inventory checked); Pitfall 5 (install collision — GSD_CF_ namespaced markers confirmed)

### Phase Ordering Rationale

- Phase 1 before everything else: `feature-aggregator.cjs` is consumed by both brainstorm auto-chain and feature mapper; building it first with full unit tests de-risks all downstream consumers.
- Phases 2a and 2b in parallel: brainstorm and architecture mode share no code paths; parallel execution is safe and reduces calendar time.
- Phase 3 after Phase 1 and after at least one successful Phase 2a brainstorm run: the feature mapper needs the aggregator working and needs a real PRD as test input to calibrate semantic deduplication.
- Phase 4 always last: integration testing can only catch regressions after all components are individually stable.
- The canonical PRD schema (`prd-format.md`) is a blocking decision for Phase 2a and must be frozen before either the brainstormer agent or the feature-aggregator AC parser is written.

### Research Flags

Phases likely needing deeper research or special attention during planning:
- **Phase 2a (Brainstorm agent design):** The two-pass UX (architecture scaffold → user confirmation → implementation pass) is a new workflow pattern not yet user-tested. The conversation depth management (max-7-questions, running ledger) needs careful agent prompt engineering. The PRD schema definition is an upfront design task that blocks all other Phase 2a work.
- **Phase 3 (Feature Map semantic deduplication):** When do two feature descriptions from different sources (PRD AC vs. code tag description) refer to the same capability? No calibration data exists until Phase 2a produces real PRDs. The `gsd-feature-mapper` agent prompt must be calibrated against real input before Phase 3 is called complete.

Phases with well-documented patterns (skip research, implement directly):
- **Phase 1 (Feature Aggregator):** Pure string processing following the exact module shape of `arc-scanner.cjs`. All patterns verified by direct codebase inspection. No unknowns.
- **Phase 2b (Architecture Mode flag):** Task() context injection — the identical mechanism used by `prototype.md` to inject the AC list into gsd-prototyper. Already proven in v1.1. Zero new mechanisms.
- **Phase 4 (Integration Validation):** Standard smoke test pattern established in v1.0/v1.1.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Zero-dep constraint verified across all lib modules; feature-aggregator.cjs follows proven arc-scanner.cjs pattern exactly; no new runtime dependencies; claude-agent-sdk version verified via npm view 2026-03-29 |
| Features | HIGH | Table stakes validated against Bolt, Kiro, spec-kit, and ChatPRD competitors via WebFetch; anti-features documented with explicit rationale; v1.1 baseline confirmed stable via direct codebase read |
| Architecture | HIGH | All patterns verified by direct inspection of 6 command files, 2 agent files, 3 lib modules, and install.js; five-layer architecture is unambiguous; build order derived from actual dependency analysis |
| Pitfalls | HIGH | v1.0/v1.1 pitfalls from prior research cycles; v1.2-specific pitfalls (21-27) derived from direct analysis of new feature interactions and boundary conditions; all prevention strategies mapped to concrete implementation steps |

**Overall confidence:** HIGH

### Gaps to Address

- **Canonical PRD schema definition:** The most important upfront design decision for v1.2. Must be written in Phase 2a and frozen before brainstormer agent or feature-aggregator AC parser is finalized. If this schema drifts between commands, Pitfall 21 is guaranteed.
- **Feature Map semantic deduplication threshold:** The `gsd-feature-mapper` agent must determine when two feature descriptions refer to the same capability. No calibration data exists until Phase 2a produces real PRDs. Treat Phase 3 as requiring calibration against Phase 2a output — do not design the prompt in isolation.
- **Two-pass architecture mode UX validation:** The architecture scaffold → user confirmation → implementation pass workflow is theoretically sound but untested as an interaction. Validate early in Phase 2b with a real greenfield project before treating as complete.
- **`arc.architecture_mode` config key:** Adding to VALID_KEYS in `config.cjs` — low risk but verify it doesn't shadow any existing key before Phase 2b commit.
- **Feature Map regeneration coupling:** The decision to couple FEATURES.md regeneration to every `extract-tags` run (not just brainstorm and prototype) must be confirmed as acceptable UX — regeneration adds latency to every extract-tags invocation. If latency is a concern, an opt-in `--update-features` flag may be preferable to always-on coupling.

---

## Sources

### Primary (HIGH confidence)

- GSD Code-First codebase direct inspection (2026-03-29): `commands/gsd/prototype.md`, `commands/gsd/iterate.md`, `commands/gsd/annotate.md`, `commands/gsd/review-code.md`, `agents/gsd-prototyper.md`, `agents/gsd-reviewer.md`, `get-shit-done/bin/lib/arc-scanner.cjs`, `get-shit-done/bin/lib/config.cjs`, `get-shit-done/bin/gsd-tools.cjs`, `bin/install.js` (lines 4204-4220) — all read directly
- `.planning/PROJECT.md` — v1.2 target features, constraints, explicit out-of-scope decisions
- `.planning/research/SUMMARY.md` (v1.1) — five-layer architecture confirmed as stable carry-forward baseline
- `.planning/config.json` — config schema with arc, workflow, phase_modes namespaces
- `npm view @anthropic-ai/claude-agent-sdk version` → `0.2.87` (2026-03-29)
- Node.js >=20 built-in API reference — `fs.readdirSync`, `String.matchAll`, `RegExp` with `/gm` flag

### Secondary (MEDIUM confidence)

- [Addy Osmani — AI-Driven Prototyping: v0, Bolt, and Lovable Compared](https://addyo.substack.com/p/ai-driven-prototyping-v0-bolt-and) — competitor feature analysis, brainstorm-to-build patterns; WebFetch verified
- [Martin Fowler — Understanding Spec-Driven-Development: Kiro, spec-kit, and Tessl](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html) — spec-as-intermediate-artifact rationale, human checkpoint patterns; WebFetch verified
- [Addy Osmani — LLM coding workflow going into 2026](https://addyosmani.com/blog/ai-coding-workflow/) — AI-first SDLC patterns; WebFetch verified in v1.1 research
- HubSpot internal code review agent analysis — two-stage reviewer/judge pattern for review output quality (cited in PITFALLS.md Pitfall 14)
- [Bolt discuss mode — Bolt Beginners Guide](https://www.nocode.mba/articles/bolt-beginners-guide) — brainstorm mode competitor reference

### Tertiary (LOW confidence — patterns only)

- [ChatPRD — conversational PRD generation](https://www.chatprd.ai/resources/using-ai-to-write-prd) — conversational question structure reference
- [AI-First SDLC Scaffold — pangon/ai-sdlc-scaffold](https://github.com/pangon/ai-sdlc-scaffold) — architecture-first prototyping patterns

---

*Research completed: 2026-03-29*
*Ready for roadmap: yes*
