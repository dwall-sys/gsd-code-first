# Feature Research

**Domain:** CLI-based autonomous coding workflow — PRD-to-prototype pipeline, test generation agent, code review agent
**Researched:** 2026-03-29
**Confidence:** HIGH (patterns verified across multiple current sources)

---

## Scope Note

This research covers ONLY the new v1.1 features. The v1.0 feature set (ARC annotation standard, tag scanner, `/gsd:prototype`, `/gsd:iterate`, `/gsd:annotate`, `/gsd:extract-plan`, per-phase modes) is treated as the stable baseline and is not re-evaluated here.

The four target features for v1.1 are:
1. **PRD-to-Prototype Pipeline** — `/gsd:prototype` reads `.planning/PRD.md`, translates acceptance criteria to `@gsd-todo` tags in the prototype
2. **ARC as default** — `arc.enabled` always `true`, remove opt-in friction
3. **Test-Agent** — new agent (`gsd-test-agent`) that writes unit/integration tests for annotated code, runs them, annotates gaps
4. **Review-Agent + `/gsd:review` overhaul** — repurpose existing command from plan-review to code review (test execution, spec compliance, code quality, actionable next steps)

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist in an autonomous prototype-and-review CLI. Missing these = product feels broken or unfinished.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| PRD file as structured input to prototype | Every autonomous coding pipeline in 2026 (Ralph, quantum-loop, n-dx) reads a spec or PRD file. Without it, `/gsd:prototype` is free-form — users cannot reliably drive it toward specific requirements. | MEDIUM | PRD at `.planning/PRD.md`. gsd-prototyper reads it alongside existing PROJECT.md/REQUIREMENTS.md. Command fails gracefully with scaffold offer if PRD missing. |
| Acceptance criteria from PRD become `@gsd-todo` tags | Users expect the prototype to reflect their requirements. If PRD ACs don't appear as `@gsd-todo` tags, the code-first loop has no connection to the PRD — it's just a scaffold. | MEDIUM | Requires gsd-prototyper to parse AC list from PRD and emit one `@gsd-todo` per criterion. Agent degrades gracefully on freeform PRDs (emits best-effort `@gsd-todo` items). |
| Test agent writes runnable tests (not stubs) | An agent billed as a "test generator" that produces placeholder stubs or non-executing tests is a known frustration. Users expect tests to run and pass. | MEDIUM | gsd-test-agent must execute the project's test command (`npm test`, `pytest`, etc.) and confirm green before completing. The test-execute-fix loop is table stakes per TiCODER research and Addy Osmani's workflow. |
| `/gsd:review` evaluates code, not plans | The existing `/gsd:review` does cross-AI plan review. After running a prototype pipeline, users calling `/gsd:review` expect code correctness evaluation — security, spec compliance, coverage. The name creates a mismatch. | MEDIUM | Repurpose `/gsd:review` for code review. Existing plan-review behavior can be preserved via `/gsd:review --plan` flag or folded into `/gsd:plan-phase`. |
| Review output includes actionable next steps | Code review tools that only list findings without recommending actions (fix, defer, accept-risk) are frustrating. Qodo, CodeAnt, and Codegen all surface next-steps as standard output. | LOW | Review-Agent output closes with a prioritized action list. Not just a findings dump. |
| Human approval gate preserved across all flows | Professional developers cite approval gates as non-negotiable before major agent actions. Ralph Loop, quantum-loop, and Addy Osmani's workflow all include human-in-the-loop checkpoints. Removing gates destroys trust. | LOW | `/gsd:iterate` already has this. PRD pipeline and review loop must respect it. `--non-interactive` as escape hatch for CI only. |
| ARC annotations always enabled by default | With ARC opt-in, new users who skip config miss the core feature. Every autonomous loop tool enforces its tracking mechanism unconditionally (Ralph's prd.json passes flag, quantum-loop's 5-state tracking). Opt-in = silent omission. | LOW | Default `arc.enabled: true` in config schema. Existing projects with explicit `arc.enabled: false` keep that setting. New installs get `true`. |

### Differentiators (Competitive Advantage)

Features that set GSD apart from Ralph, quantum-loop, Addy Osmani's workflow, and other autonomous coding pipelines.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| PRD ACs become machine-readable `@gsd-todo` tags | Competitors treat PRD as prose context fed to the agent. GSD translates each acceptance criterion into an `@gsd-todo` tag embedded in prototype code. This makes the PRD iterable via `/gsd:iterate` — each AC is directly executable. No competitor does this end-to-end. | MEDIUM | Requires a structured AC section in the PRD (lightweight template). gsd-prototyper parses it. The result: PRD in → annotated code out → iterate executes the ACs. Closes the loop uniquely. |
| Test-Agent annotates untested paths with `@gsd-risk` | Generic test agents write tests and stop. GSD's Test-Agent annotates code paths that are untested or hard to test with `@gsd-risk` tags. These feed into CODE-INVENTORY.md automatically — risk is visible in planning artifacts, not silently ignored. | MEDIUM | `@gsd-risk` is already in the ARC standard. No new tooling needed — tag scanner picks them up. Agent design choice only. |
| Two-stage review: spec compliance before code quality | quantum-loop's insight: "Stage 2 never runs if Stage 1 fails." For GSD: verify that PRD acceptance criteria are met before evaluating code quality. Surfaces requirements gaps before wasting review cycles on style/security in code that doesn't meet the spec. | MEDIUM | Stage 1: check each PRD AC against code/tests. Stage 2: correctness, security, maintainability. Clear output sections. Agent design — no extra tooling. |
| ARC always-on removes "what mode am I in" cognitive overhead | With ARC opt-in, users must remember to configure it. Making ARC always-on means the `/gsd:iterate` step 4 branch (`config-get arc.enabled`) is removed — always spawns `gsd-arc-executor`. Simpler mental model, simpler code path. | LOW | Also simplifies future documentation — no mode explanation needed. |
| Review findings structured for future `--fix` chaining | If review output follows a consistent Markdown schema (defined sections, machine-parseable action items), a future `--fix` flag can pipe review findings into `/gsd:iterate` as `@gsd-todo` tags. No competitor closes this loop. | LOW (design only now) | Design the output schema now even if `--fix` is deferred. Costs nothing at design time; retroactively structuring output is expensive. |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem productive but create problems in this domain.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Fully autonomous PRD-to-tests-to-review with no human checkpoint | Maximum productivity, zero interruptions | Agents hallucinate APIs, miss domain edge cases, produce plausible-but-wrong code. Shipping 1000-line agent PRs without review is a documented anti-pattern (multiple 2026 sources). One wrong prototype decision compounds through tests and review. | Keep approval gates. Make each step fast (show diff, await Y/N). `--non-interactive` for CI only. |
| Test agent targeting 100% coverage | Sounds thorough | Produces tests that mock behavior instead of testing it, brittle tests, and coverage theater — all green, zero real safety net. This is the #1 anti-pattern in automated test generation (TiCODER research, Codepipes anti-patterns). | Target tests for `@gsd-risk` annotated paths and critical paths first. Accept < 100% coverage with honest `@gsd-risk` annotation rather than fake 100%. |
| Single mega-agent for PRD + prototype + tests + review | Simpler one-command interface | Context window exhaustion, role confusion, inability to restart at failed step, poor separation of concerns. Single-agent approaches for multi-step complex pipelines consistently underperform specialized agents (validated by ChatDev, MetaGPT, quantum-loop research). | Specialized agents: gsd-prototyper, gsd-test-agent, gsd-review-agent. Chain via commands with checkpoints. |
| Free-form PRD with no structure requirement | Zero friction for users | Non-deterministic prototype quality. Different prose styles produce radically different output quality, making the tool feel unreliable. Agents cannot reliably extract acceptance criteria from unstructured paragraphs. | Lightweight PRD template (problem statement + acceptance criteria list). Ship as scaffold. Agent handles prose outside the template gracefully — template defines the minimum structure. |
| Parallel test generation across all files simultaneously | Faster test coverage | Duplicate test coverage, conflicting test fixtures, race conditions in shared test infrastructure (setup/teardown files). Coordination overhead exceeds parallelism benefit for test generation specifically. | Sequential test generation with file-level isolation. Fast enough; avoids coordination complexity entirely. |
| Auto-commit after prototype or test generation | Convenience, less manual git work | Autonomous commits without review is the pattern that erodes developer trust in AI tooling. Codex CLI and quantum-loop both explicitly gate on human approval before any git operations. | Show what would be committed. Let the developer decide. At most, stage files — never commit without explicit user action. |

---

## Feature Dependencies

```
[.planning/PRD.md]
    └──required-by──> [PRD-to-Prototype Pipeline]
                          └──gsd-prototyper reads PRD
                          └──produces──> [Prototype with @gsd-todo tags from ACs]
                                             └──feeds──> [/gsd:iterate (existing, unchanged)]
                                             └──feeds──> [gsd-test-agent]

[Prototype / existing codebase with @gsd-tags]
    └──required-by──> [gsd-test-agent]
                          └──writes test files
                          └──runs test command
                          └──emits @gsd-risk tags on untested paths
                          └──feeds──> [CODE-INVENTORY.md (via extract-plan)]

[Runnable test suite]
    └──required-by──> [gsd-review-agent Stage 2]
                          (Stage 1 can run without tests; Stage 2 needs test execution)

[ARC always-on (arc.enabled: true default)]
    └──simplifies──> [/gsd:iterate step 4] (removes config-get branch, always uses gsd-arc-executor)
    └──enables──> [gsd-review-agent] (can assume @gsd-risk tags exist; no "ARC might be off" guard)

[Review output (REVIEW-CODE.md, structured Markdown)]
    └──enables──> [future --fix flag] (pipe findings into iterate as @gsd-todo tags)
    └──must-not-conflict-with──> [existing REVIEWS.md] (plan review output from original /gsd:review)
```

### Dependency Notes

- **PRD-to-Prototype requires a PRD file:** Command must fail gracefully with a scaffold offer (`/gsd:prototype --init-prd`) if `.planning/PRD.md` is missing. Hard failure without guidance abandons users.
- **Test-Agent requires code to exist:** Can run on prototype output OR on existing codebases (brownfield). Both paths valid. Agent reads `@gsd-todo` and `@gsd-risk` tags as priority hints for what to test first.
- **Review-Agent Stage 2 requires a test runner:** If no test command is detected (`npm test`, `pytest`, `make test`), Stage 2 should document the absence as a `@gsd-risk` rather than silently skipping execution. Do not fail silently.
- **ARC always-on simplifies iterate:** Removing the `config-get arc.enabled` branch in `/gsd:iterate` step 4 is a clean-up task enabled by making ARC the default. Track as a sub-task of "ARC as default."
- **Review output file naming:** Existing `/gsd:review` produces `REVIEWS.md` (plan review). New code review must produce a different file — `REVIEW-CODE.md` — to prevent overwriting the existing plan-review artifact. This is a hard naming constraint.

---

## MVP Definition

### Launch With (v1.1)

Minimum viable product for this milestone — what validates the "PRD in, prototype out, tested and reviewed" loop.

- [ ] **ARC always-on** — Config default change + remove branch in `/gsd:iterate` step 4. Low complexity, high payoff. Unlocks everything downstream cleanly.
- [ ] **PRD-to-Prototype pipeline** — Update gsd-prototyper to read `.planning/PRD.md`, parse acceptance criteria, emit `@gsd-todo` tags per AC. Update `/gsd:prototype` command to pass PRD path as context. Add graceful missing-PRD handling.
- [ ] **gsd-test-agent + `/gsd:add-tests`** — New agent that writes runnable tests, executes them, annotates untested paths with `@gsd-risk`. Wire into existing `/gsd:add-tests` command (currently exists — verify if stub or implemented).
- [ ] **gsd-review-agent + `/gsd:review` overhaul** — Replace plan-review behavior with two-stage code review (spec compliance then code quality). Output to `REVIEW-CODE.md`. Preserve existing plan-review as `/gsd:review --plan`. Actionable next-steps section required.

### Add After Validation (v1.1+)

- [ ] **Review-to-iterate chain (`--fix` flag)** — Pipe structured review findings into `/gsd:iterate` as `@gsd-todo` tags. Trigger: users manually copying review findings into iterate more than twice.
- [ ] **PRD template scaffolding (`/gsd:prototype --init-prd`)** — Generate `.planning/PRD.md` template with problem statement + AC list sections. Trigger: users reporting confusion about PRD format.
- [ ] **`@gsd-coverage` tag type** — Surface test coverage gaps in CODE-INVENTORY.md as a new tag type. Trigger: Test-Agent adopted and coverage visibility becomes a planning need.

### Future Consideration (v2+)

- [ ] **Parallel test generation** — Defer: adds coordination complexity, not needed for single-developer workflow.
- [ ] **Automated CI/CD integration** — Defer until there's clear user demand for unattended pipeline runs.
- [ ] **Multi-language test runner auto-detection beyond Node.js** — Start with `npm test` (known project context). Add Python/Go later.

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| ARC always-on | HIGH | LOW | P1 |
| PRD-to-Prototype pipeline | HIGH | MEDIUM | P1 |
| gsd-test-agent + test command | HIGH | MEDIUM | P1 |
| `/gsd:review` overhaul — code review | HIGH | MEDIUM | P1 |
| Two-stage review gate (spec then quality) | MEDIUM | LOW (agent design only) | P2 |
| `@gsd-risk` tags from Test-Agent | MEDIUM | LOW (agent design, tag exists) | P2 |
| Structured review output schema for future `--fix` | MEDIUM | LOW (design choice now) | P2 |
| Review-to-iterate chain (`--fix`) | MEDIUM | HIGH | P3 |
| PRD template scaffolding | LOW | LOW | P3 |

**Priority key:**
- P1: Must have for v1.1 launch
- P2: Should have, include in v1.1 if capacity permits
- P3: Defer to v1.1+ or v2

---

## Competitor Feature Analysis

These tools define the feature expectations users bring to autonomous coding pipelines. None are direct GSD competitors (no ARC annotations, no code-first philosophy), but they set the baseline users expect.

| Feature | Ralph Loop | quantum-loop | Addy Osmani Workflow | GSD v1.1 Approach |
|---------|------------|--------------|----------------------|-------------------|
| PRD input format | prd.json with user stories + passes flag | spec file → structured tasks JSON | Prose brainstorm → structured plan | `.planning/PRD.md` Markdown (lightweight, no JSON) |
| Prototype generation | Iterative loop until all PRD items complete | DAG-based parallel story execution | Sequential task execution | Single-shot prototype with ARC tags, then iterate |
| AC → task translation | passes flag per story in prd.json | 5-state story tracking in quantum.json | Manual task breakdown | `@gsd-todo` tag per AC, embedded in code |
| Test integration | Test runner called post-implementation; loop retries on failure | TDD: tests before implementation | Write tests → run → fix loop | Test-Agent post-prototype; runs tests; annotates gaps with `@gsd-risk` |
| Code review | Not built-in | Two-stage (spec then code quality) | Human line-by-line + secondary AI session | Review-Agent: two-stage output in REVIEW-CODE.md |
| Approval gate | No explicit gate; loop runs until prd.json is complete | Human review gate between stages | Explicit human review before merge | Preserved: human approval before each major step |
| State persistence | prd.json + progress.txt + git history | quantum.json state machine | Git + task list | @gsd-tags embedded in code + CODE-INVENTORY.md |

**Key GSD differentiation:** Competitors track state in external JSON/config files separate from the code. GSD embeds state (as ARC tags) directly in the code. The code IS the source of truth — no sync problem between code and state file. This is the core architectural bet of the entire fork.

---

## Sources

- [quantum-loop — spec-driven with DAG execution and two-stage review gates](https://github.com/andyzengmath/quantum-loop) — HIGH confidence (WebFetch confirmed architecture)
- [Ralph Loop — PRD-driven autonomous coding loop](https://github.com/snarktank/ralph) — MEDIUM confidence (WebSearch + Ralph Wiggum review article verified)
- [Addy Osmani — LLM coding workflow going into 2026](https://addyosmani.com/blog/ai-coding-workflow/) — HIGH confidence (WebFetch confirmed table stakes patterns)
- [TiCODER — test-driven interactive code generation research (Penn)](https://www.seas.upenn.edu/~asnaik/assets/papers/tse24_ticoder.pdf) — MEDIUM confidence (abstract verified via search)
- [Agentic Coding Trends Report 2026 — Anthropic](https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf) — HIGH confidence (official Anthropic publication)
- [AI Code Review Tools: 8 Options for the Agent Era — Codegen](https://codegen.com/blog/ai-code-review-tools/) — MEDIUM confidence (WebSearch)
- [Code Review in the Age of AI — Addy Osmani](https://addyo.substack.com/p/code-review-in-the-age-of-ai) — MEDIUM confidence (WebSearch)
- [Software Testing Anti-Patterns — Codepipes](https://blog.codepipes.com/testing/software-testing-antipatterns.html) — HIGH confidence (well-known reference, multiple corroborating sources)
- [AI Agent Anti-Patterns Part 1 — Allen Chan, March 2026](https://achan2013.medium.com/ai-agent-anti-patterns-part-1-architectural-pitfalls-that-break-enterprise-agents-before-they-32d211dded43) — MEDIUM confidence (WebSearch, recent)
- [How to Do Code Reviews in the Agentic Era — Google Cloud Community](https://medium.com/google-cloud/how-to-do-code-reviews-in-the-agentic-era-0b6584700f47) — MEDIUM confidence (WebSearch)
- [The State of AI Coding Agents 2026 — Dave Patten](https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a) — MEDIUM confidence (WebSearch)
- Existing codebase: `/commands/gsd/prototype.md`, `/commands/gsd/review.md`, `/commands/gsd/iterate.md` — HIGH confidence (direct read)

---

*Feature research for: gsd-code-first v1.1 autonomous prototype and review loop*
*Researched: 2026-03-29*
