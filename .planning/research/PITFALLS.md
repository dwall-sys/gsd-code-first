# Pitfalls Research

**Domain:** CLI framework fork with code annotation system and agent orchestration
**Researched:** 2026-03-29
**Confidence:** HIGH (fork compatibility, regex parsing, agent integration) | MEDIUM (autonomous loop prevention, test generation quality)

---

> **Note on scope:** Pitfalls 1-11 (below line) were written for v1.0. This document extends them with pitfalls specific to the **v1.1 milestone**: autonomous agent loops, test generation, review agents, and PRD-to-prototype pipeline.

---

## Critical Pitfalls

Mistakes that cause rewrites, major compatibility breaks, or adoption failure.

---

### Pitfall 1: Fork Divergence — Touching Upstream Files

**What goes wrong:** The fork modifies files that upstream GSD also modifies — `package.json` scripts, `gsd-executor.md`, `gsd-planner.md`, `install.js`, `gsd-tools.cjs`. When upstream releases changes, every modified file becomes a merge conflict site. Over time, upstream sync is abandoned because the cost is too high.

**Why it happens:** "Just a small tweak" to an existing agent prompt feels low-risk. But each tweak is a permanent fork point. The cumulative weight of small invasive changes makes upstream rebasing impossible by month 3.

**Consequences:**
- Security patches from upstream never land in the fork
- Upstream feature improvements must be manually cherry-picked
- The fork becomes an island, negating the "upstream mergeability" constraint
- Users of the fork miss upstream fixes to commands they rely on

**Warning signs:**
- Git diff against upstream shows modifications to files that aren't new
- Merge conflicts appear in non-ARC files during upstream sync attempts
- Changelog entries reference changes to `gsd-executor.md` or `gsd-planner.md`

**Prevention:**
- Treat all existing upstream files as read-only. Never edit them; wrap them instead.
- Add new agents (`gsd-prototyper.md`, `gsd-annotator.md`, `gsd-code-planner.md`) as net-new files
- The "Modified gsd-executor with ARC comment obligation" requirement in PROJECT.md is HIGH RISK — implement it as a new `gsd-arc-executor.md` that imports executor behavior, not as a patch to `gsd-executor.md`
- Maintain a file-level inventory: "touched by fork" vs. "upstream-owned"
- Run `git diff upstream/main -- <file>` as a CI check on all upstream-owned files

**Phase to address:** Phase 1 (project setup) — establish the no-touch policy before any code is written.

---

### Pitfall 2: Regex Tag Extraction False Positives and False Negatives

**What goes wrong:** The `@gsd-tag` regex scanner extracts tags from inside string literals, template literals, and URL paths (e.g., `"http://example.com/@gsd/thing"`), producing garbage in `CODE-INVENTORY.md`. It also misses tags in multiline `/* */` blocks when the pattern doesn't account for newlines correctly, silently dropping planning data.

**Why it happens:** JavaScript regex cannot model JavaScript syntax. The same character sequence appears in comments, strings, template literals, and regex literals. A pattern matching `// @gsd-context:` will fire on `const msg = "// @gsd-context: not a tag"`.

**Consequences:**
- `CODE-INVENTORY.md` is polluted with false extractions, undermining trust in the tool
- Real tags in block comments are missed, creating invisible planning gaps
- Developers who discover extraction errors stop trusting the output and stop annotating

**Warning signs:**
- Tag counts from scanner don't match manual grep on a known file
- Tags appear in inventory that were never written by a developer
- Tags inside JSDoc blocks are missing from output

**Prevention:**
- Skip lines where `@gsd-` appears inside a string context: require the tag to appear in a line starting with `//`, `*`, or `#` after optional whitespace
- Pattern: `/^\s*(?:\/\/|\/\*|\*|#)\s*@gsd-(\w+):/m` — anchors to comment-opening tokens
- Test the scanner against a fixture file with deliberate edge cases: URL in string, JSDoc, multiline block comment, template literal with `@` character
- Document the extraction rules in the ARC standard so developers know what is and is not a valid tag location

**Phase to address:** Phase 1 (scanner implementation) — build the test fixture at the same time as the scanner.

---

### Pitfall 3: ARC Tag Standard Churn — The Annotation Compatibility Problem

**What goes wrong:** The `@gsd-tags` annotation standard is defined early, shipped, developers annotate their codebases, then the standard evolves (renaming `@gsd-todo` to `@gsd-task`, adding required fields, changing format). Every existing annotated codebase is now broken or produces degraded output. Developers have to re-annotate.

**Why it happens:** Tag standards are designed before real usage reveals what information is actually needed by the planner agent. The first iteration is always incomplete. The temptation is to fix it by changing the standard rather than extending it.

**Consequences:**
- Re-annotation cost is high and resented — adoption drops
- The scanner produces mixed results from old and new tag formats
- Trust in the tool's stability collapses

**Warning signs:**
- A tag name or format change is proposed before the first production use
- The `CODE-INVENTORY.md` output has empty fields that nobody fills in
- Planner agent prompts reference tag fields that the scanner doesn't emit

**Prevention:**
- Treat the ARC standard as a versioned schema from day one: `@gsd-context:` is version 1.0 and cannot change, only new tags can be added
- Design the scanner to be lenient on optional fields: if a tag has fewer fields than expected, populate with defaults not errors
- Run at least one real annotation session on a non-trivial codebase before freezing the standard
- The standard document should explicitly state: "Tag names and required fields are stable. New optional fields may be added."

**Phase to address:** Phase 1 (ARC standard definition) — stabilize before implementation, not after.

---

### Pitfall 4: Agent Prompt Drift — Planner and Prototyper Behave Differently Over Time

**What goes wrong:** `gsd-prototyper.md` and `gsd-code-planner.md` are written once during development, but as the framework evolves, the prompts become inconsistent with each other. The prototyper adds tags the planner doesn't know about. The planner references extraction fields the scanner doesn't produce. The system degrades silently.

**Why it happens:** Agent prompts are treated like documentation — written once, rarely updated. But they are executable contracts between components. When one component changes, the prompt of its downstream consumers must change too.

**Consequences:**
- The `iterate` command produces plans based on incomplete tag data
- The `code-planner` agent hallucinates structure because its input format has drifted from what the scanner produces
- Debugging is hard because the failure is in behavior, not in a crash

**Warning signs:**
- The planner output references section headings not present in `CODE-INVENTORY.md`
- The prototyper is adding tags not listed in the ARC standard
- The `iterate` command produces different quality output on different codebases without obvious cause

**Prevention:**
- Define a canonical `CODE-INVENTORY.md` schema and treat it as a contract: scanner writes it, planner reads it
- Include a schema version field in `CODE-INVENTORY.md` so agents can detect format mismatches
- Agent prompts should include the exact field names they expect from their inputs — make the coupling explicit rather than implicit
- When updating any component, explicitly check: "which agent prompts depend on this component's output format?"

**Phase to address:** Phase 2 (agent implementation) — define the schema contract before writing either the scanner or the planner agent.

---

### Pitfall 5: `npx gsd-code-first@latest` Install Collision with `npx get-shit-done-cc@latest`

**What goes wrong:** Users install both the fork and the upstream. The `install.js` in both writes to the same `.claude/` directory locations. One installation clobbers the other's agent files or config markers. Users experience confusing behavior where commands from one tool call agents from the other.

**Why it happens:** The upstream installer uses file markers (`# GSD Agent Configuration — managed by get-shit-done installer`) without namespace scope. If the fork uses identical markers, both installers claim ownership of the same sections.

**Consequences:**
- Running `npx gsd-code-first@latest` after `npx get-shit-done-cc@latest` silently overwrites upstream agents
- Re-running upstream install after fork install removes fork-specific agents
- Users cannot safely have both installed

**Warning signs:**
- `install.js` in the fork uses the same `GSD_CODEX_MARKER` constant as upstream
- No namespacing of agent files (both write to `.claude/agents/gsd-executor.md`)
- Users report that re-running the upstream installer removes fork agents

**Prevention:**
- Namespace all fork-specific marker constants: `GSD_CF_CODEX_MARKER` (CF = Code-First)
- Fork-specific agents use a distinct prefix or subdirectory that the upstream installer never touches
- Installation docs must warn: "This fork replaces the upstream installation" — do not support concurrent installation
- In `install.js`, detect upstream installation and warn rather than silently overwriting

**Phase to address:** Phase 1 (installer update).

---

### Pitfall 6: The `iterate` Command — Approval Step Blocking in Headless/Scripted Contexts

**What goes wrong:** The `iterate` command is designed as `extract-tags → code-planner → approval → executor`. The approval step requires human input. When users run this in CI, scripts, or other headless contexts, it hangs indefinitely. Users bypass the approval by killing the process mid-execution, leaving state corrupted.

**Why it happens:** Interactive approval flows work in the terminal but are not designed for automation. The original GSD framework has faced this problem (see recent headless prompt overhaul in the commit history: `feat: Headless prompt overhaul — SDK drives full lifecycle without interactive patterns`).

**Consequences:**
- CI pipelines fail or hang
- Developers who want scripted iteration cannot use the tool's most valuable command
- State files left in partial states cause subsequent runs to fail with confusing errors

**Warning signs:**
- The approval step uses `readline` or similar interactive input without a `--yes`/`--non-interactive` flag
- No timeout or default-deny behavior on approval prompts
- No documentation of what happens if the process is killed during approval

**Prevention:**
- Follow the upstream's headless prompt overhaul pattern — the framework already solved this
- Add `--non-interactive` flag to `iterate` that skips approval (outputs the plan and exits with 0 for piped use)
- Alternatively: separate approval into its own command so `iterate --no-approve` generates the plan file without executing
- Always provide a `--yes` auto-approve flag for scripted use cases

**Phase to address:** Phase 2 (iterate command implementation).

---

### Pitfall 7: Context Window Saturation in the `iterate` Workflow

**What goes wrong:** The `iterate` command feeds the full codebase scan output (`CODE-INVENTORY.md`) into the `gsd-code-planner` agent. For large codebases with many annotated files, this document becomes very large, consuming most of the agent's context window before the planning task even begins.

**Why it happens:** Agent context windows are finite. A codebase with 200 annotated files producing 50 lines of inventory each results in a 10,000-line input document. Tool definitions, system prompts, and conversation history compound this.

**Consequences:**
- The planner produces truncated or lower-quality plans on large codebases
- The tool works well in demos (small codebases) but degrades in real projects
- Adoption stalls because the tool "doesn't scale"

**Warning signs:**
- `CODE-INVENTORY.md` exceeds 1,000 lines on real projects
- Planner output quality varies suspiciously based on codebase size
- Claude API response times increase dramatically on large inventory files

**Prevention:**
- The `extract-plan` command should support a `--phase` or `--scope` filter to extract only tags relevant to the current iteration
- `CODE-INVENTORY.md` should be structured by phase/component so the planner can be given a slice
- Design the inventory format to be chunked: the planner reads a summary section first, then requests specific sections
- Set a soft limit on inventory size and warn when it's exceeded

**Phase to address:** Phase 2 (extract-plan command design), before building the planner agent.

---

### Pitfall 8: Developer Adoption Failure — Annotation Feels Like Overhead

**What goes wrong:** The ARC annotation system requires developers to add structured comments as they prototype. Developers under time pressure write the prototype, ship it, and never annotate. The `annotate` command (retroactive annotation) is used as the escape hatch. But retroactive annotation by the `gsd-annotator` agent produces lower-quality tags than annotations written during development (because the agent infers intent rather than capturing it).

**Why it happens:** Adding annotations during coding interrupts flow. Developers optimize for immediate output, not downstream planning quality. Tools that add cognitive load during coding face systematic avoidance.

**Consequences:**
- The `iterate` command operates on weak planning data, producing mediocre plans
- The differentiated value of "code IS the plan" is lost — it becomes just another planning layer
- The tool is used only for post-hoc documentation, not as a live workflow tool

**Warning signs:**
- Most annotation sessions use `annotate` (retroactive) rather than annotations added during prototyping
- The `gsd-prototyper` agent is rarely used — developers write prototypes by hand without ARC tags
- User feedback: "the annotations feel like busy work"

**Prevention:**
- The `gsd-prototyper` agent is the key forcing function — it adds ARC tags automatically as it generates code, making annotation the zero-cost path
- Make the annotation density visible: `extract-plan --stats` showing coverage per file rewards annotating behavior
- Start with fewer required tags: `@gsd-context` and `@gsd-decision` are high-value and low-effort; `@gsd-constraint` and `@gsd-risk` can be optional
- The ARC standard should have a "minimal annotation" tier that is sufficient for planning, and an "full annotation" tier for documentation quality

**Phase to address:** Phase 1 (ARC standard design), Phase 2 (prototyper agent implementation).

---

## v1.1 Critical Pitfalls

Pitfalls specific to the v1.1 milestone: autonomous prototype pipeline, test agent, review agent, and PRD parsing.

---

### Pitfall 12: Autonomous Agent Loop Divergence — The Prototype Never Converges

**What goes wrong:** The PRD-to-prototype pipeline runs `gsd-prototyper` in an autonomous loop — parse PRD, generate code, evaluate output, iterate. Without hard termination conditions, the loop diverges: the agent reinterprets requirements differently on each pass, produces contradictory implementations, and never reaches a stable output. Each iteration overwrites the previous one, so there is no recoverable state.

**Why it happens:** LLMs lack implicit memory of action history within a single task. The model does not "know" it already tried an approach unless the failed attempt is explicitly in its context. Vague PRD sections produce ambiguous acceptance criteria — the agent can't recognize when it's done because "done" isn't defined. Small misunderstandings compound: a loop that starts 5% off target diverges further with each iteration rather than converging.

**Consequences:**
- Prototype output quality degrades rather than improves across iterations
- Conflicting implementations accumulate across files, producing a codebase that can't run
- Users cannot predict when the loop will finish — it appears to run forever
- Token costs and wall-clock time grow unboundedly

**Warning signs:**
- The prototype loop runs more than 3 iterations without the agent reporting completion
- Successive `PROTOTYPE-LOG.md` files list contradictory design decisions
- The same requirement appears as "implemented" in one file and "todo" in another
- Agent output contains phrases like "revising approach" or "reconsidering architecture" more than once

**Prevention:**
- Hard cap iterations at a fixed maximum (3 is a reasonable default for prototype loops; configurable via `arc.max_prototype_iterations`)
- Define acceptance criteria before the loop starts, not inside the loop: PRD-to-prototype must emit a checklist of "done" signals that the evaluator checks after each pass
- Track what was attempted in a structured log (not just narrative) so the agent can see its own history: `PROTOTYPE-LOG.md` should include an `## Attempts` section with per-iteration outcome
- When the iteration cap is hit, stop and present what was produced rather than failing silently
- Separate "prototype" (one-shot scaffold) from "iterate" (approval-gated refinement) — the autonomous loop is for PRD parsing and initial scaffold only, not open-ended improvement

**Phase to address:** PRD-to-Prototype Pipeline phase — define termination logic before building the loop, not after.

---

### Pitfall 13: Test Agent Producing Tests That Always Pass — False Coverage Confidence

**What goes wrong:** The test agent generates tests that technically execute and report green, but do not verify the behavior they claim to test. Common patterns: assertions that test the mock rather than the implementation (`expect(mockFn).toHaveBeenCalled()` without testing the result), tests that always pass because the expected value is derived from the same implementation under test, and snapshot tests that capture incorrect behavior and lock it in.

**Why it happens:** LLMs optimize for producing syntactically valid, plausibly structured test code that matches the patterns in their training data. They do not have a ground-truth model of the implementation's intended behavior — only its actual behavior. The model generates tests that are consistent with the code it sees, not tests that would catch bugs in that code. Research confirms LLMs are overconfident about test coverage: they report "tests pass" as a success signal without distinguishing "passes because behavior is correct" from "passes because test is vacuous."

**Consequences:**
- Green test suite gives false assurance that the prototype is correct
- The review agent reads test results as validation of implementation quality — if tests are vacuous, review is compromised
- Bugs that the tests should catch survive into production
- Developers stop trusting the test agent output after the first false negative

**Warning signs:**
- All generated tests pass on the first run against unreviewed prototype code (real tests typically fail initially against scaffold implementations)
- Test assertions check mock call counts rather than return values
- Expected values in assertions are computed by calling the function under test rather than hardcoded
- No test in the generated suite would fail if the function body were replaced with `return null`
- Test descriptions say "should work" or "should not throw" without specifying behavior

**Prevention:**
- The test agent prompt must explicitly require negative tests: "Write at least one test that would fail if the function returned the wrong type, and at least one test that would fail if a required field were missing"
- Require the test agent to write the expected value before calling the function: "State what the function should return, then assert it" — prevents deriving expected from actual
- Run tests against a deliberately broken implementation as a validation step: if all tests still pass when a key function is stubbed to return `null`, the tests are vacuous
- Add a "test audit" step: after generation, spawn a second agent pass that reviews each test assertion and flags ones that would pass on a trivially wrong implementation
- Use RED-GREEN discipline: agent writes tests against the scaffold (which should have stub implementations) — tests must fail initially, then pass after implementation

**Phase to address:** Test-Agent phase — build the test audit step as a required gate, not an optional enhancement.

---

### Pitfall 14: Review Agent Output That Is Verbose and Low-Signal

**What goes wrong:** The review agent produces multi-page reports that developers skip entirely. Every file gets the same generic feedback ("consider adding error handling," "this function could be more modular"), making every review look identical regardless of actual quality. Developers habituate to ignoring it within a week. Alternatively, the review is too shallow — it summarizes what the code does rather than evaluating whether it does it correctly.

**Why it happens:** Without explicit constraints on output structure and a quality filter on the feedback itself, LLMs default to comprehensive coverage (verbosity) or surface-level description (shallowness). Both are failure modes from the same root: no forcing function for prioritization. HubSpot's internal analysis of their code review agent identified this as the primary adoption failure mode and solved it by introducing a Judge Agent that filters output for succinctness, accuracy, and actionability.

**Consequences:**
- Review output is ignored — no actual quality improvement
- The `/gsd:review` command is used once or twice then abandoned
- Developers route around the review step, undermining the pipeline's verification gate

**Warning signs:**
- Review output exceeds 2 pages for a single-file change
- Multiple review runs on the same unchanged code produce substantially different feedback
- Feedback items contain no file/line references — they could apply to any codebase
- User skims the review and says "yeah, looks good" without reading it

**Prevention:**
- Constrain review output format: maximum N actionable items per file, each item must include file path, line range, severity (HIGH/MEDIUM/LOW), and a concrete action ("change X to Y") not a vague suggestion
- Implement a two-stage review: a Reviewer agent generates raw feedback, a Judge agent filters it to only items that are Succinct (one sentence), Accurate (technically correct), and Actionable (can be applied directly). This pattern is validated by HubSpot's production experience.
- Distinguish review types: "test coverage review" (did tests cover the right paths?), "implementation review" (does the code match the PRD intent?), "ARC annotation review" (are tags accurate after implementation?). Each is a separate, focused review, not one mega-review.
- Cap total review output: if the reviewer produces more than 10 items, the judge must reduce to the top 5 by severity before presenting to the user

**Phase to address:** Review-Agent phase — define the two-stage architecture and output contract before building the review prompt.

---

### Pitfall 15: `/gsd:review` Command Name Collision with New Review Agent

**What goes wrong:** The existing `/gsd:review` command performs cross-AI peer review of phase plans using external CLIs (Gemini, Codex, Claude in a separate session). The v1.1 "Review-Agent" feature adds code quality review — test execution, evaluation, manual verification, next steps. If both are named `/gsd:review`, users run the wrong one, get confusing output, and cannot predict which review type will execute.

**Why it happens:** The name "review" is overloaded. Plan review and code review are both called "review" in natural language but serve completely different purposes at different workflow stages. The existing command is explicitly documented as cross-AI peer review for plans, not code. Adding a second command with the same name or an overlapping alias creates silent command shadowing.

**Consequences:**
- Running `/gsd:review` after `iterate` invokes the plan reviewer, not the code reviewer
- The plan reviewer tries to read PLAN.md files and finds prototype code — fails or produces nonsense
- Users cannot discover which review command they need without reading both command files
- The new review agent effectively replaces the old command in the user's mental model, but the old command still exists and conflicts

**Warning signs:**
- The new review command file is placed at `.claude/commands/gsd/review.md` — same path as the existing command
- Both commands accept `--phase N` as an argument
- The help output lists "review" once but both commands are registered
- Users report "review gives me plan feedback when I expected code feedback"

**Prevention:**
- Name the new command distinctly: `/gsd:review-code` or `/gsd:verify-prototype` — never reuse the name `/gsd:review`
- The existing `/gsd:review` is a plan-review tool; document this distinction explicitly in both command files
- The new review command file must be placed at a different path: `commands/gsd/review-code.md` or `commands/gsd/verify-prototype.md`
- Add a disambiguation note to both commands: "For cross-AI plan review, use `/gsd:review`. For code quality and test verification, use `/gsd:review-code`."
- Update `/gsd:help` to show both commands with their distinct purposes

**Phase to address:** Review-Agent phase — name the command before writing it, confirm no collision with existing `review.md`.

---

### Pitfall 16: PRD Parsing Brittleness — Format Variations Break the Pipeline

**What goes wrong:** The PRD-to-prototype pipeline assumes a specific PRD structure (sections for Goals, Requirements, Acceptance Criteria, etc.). Real PRDs are inconsistent: some users write prose paragraphs, some use bullet lists, some embed requirements in tables, some omit acceptance criteria entirely. The pipeline silently misses requirements it can't parse, producing a prototype that implements 60% of the PRD with no indication that 40% was skipped.

**Why it happens:** AI cannot infer from omission. A parser that expects `## Acceptance Criteria` and finds `## Success Metrics` treats the section as absent. Free-form prose embeds requirements in running text that structural parsers miss entirely. The parser was designed against a specific PRD template and tested only with that template.

**Consequences:**
- Prototype is incomplete, but neither the user nor the pipeline knows which requirements were missed
- The iterate loop works on partial requirements, producing a confidently incomplete implementation
- Users discover missing features only after execution, requiring expensive rework

**Warning signs:**
- `PROTOTYPE-LOG.md` lists fewer requirements than the PRD contains
- Requirements mentioned in PRD prose paragraphs (not in structured lists) don't appear in `CODE-INVENTORY.md`
- The pipeline runs successfully on the example PRD but fails on a real PRD with different structure

**Prevention:**
- The PRD parser must emit a "requirements found" list alongside "requirements used" — the user must approve the parsed list before prototype generation begins. Silent discards are never acceptable.
- Use semantic extraction rather than structural parsing: instruct the agent to find all requirements regardless of format, then normalize them, rather than mapping a fixed section structure
- Provide a canonical PRD template and make it the documented input format for the pipeline — but gracefully degrade for non-template PRDs with an explicit warning: "N requirements may have been missed because your PRD doesn't follow the expected format"
- Test the parser against at least 3 different PRD formats before shipping: the canonical template, a prose-only document, and a table-based spec

**Phase to address:** PRD-to-Prototype Pipeline phase — build and validate the requirements extraction step before the scaffold generation step. Never combine parsing and generation into a single agent pass.

---

### Pitfall 17: ARC-as-Default Breaks Existing Non-ARC Workflows Silently

**What goes wrong:** Setting `arc.enabled` to `true` by default changes the behavior of the `iterate` command for all users, including those currently using the traditional (non-ARC) workflow. Users who have not annotated their code now get `gsd-arc-executor` spawned instead of `gsd-executor` — an executor that requires ARC tags in its output. Without tags, the arc-executor produces degraded output or fails, and the user has no idea why their previously working `iterate` command broke.

**Why it happens:** "ARC as default" is a configuration change with behavioral consequences that are not surfaced to the user. The config key `arc.enabled` is read silently at runtime; there is no prompt or warning that existing behavior is changing.

**Consequences:**
- Users who upgrade to v1.1 and run `/gsd:iterate` on an unannotated codebase get arc-executor behavior without knowing it
- The arc-executor references ARC tag obligations in its output, confusing users who don't know what ARC is
- Support burden increases as users report "iterate is broken" after upgrading

**Warning signs:**
- `arc.enabled` is set to `true` in the default config written by the installer
- The `iterate` command does not print which executor it's using at runtime
- No migration path for users upgrading from v1.0 who had `arc.enabled: false`

**Prevention:**
- Do not change `arc.enabled` default in the installer — existing users who installed v1.0 have `arc.enabled: false` in their config. Changing the default only affects new installs.
- For new installs (v1.1+), set `arc.enabled: true` as the default — acceptable because new users have no existing unannotated workflow
- For existing installs (upgrade path), print a one-time migration notice: "ARC is now the default mode. Your current setting (arc.enabled: false) is preserved. Run `/gsd:set-mode` to opt in."
- The `iterate` command must always print which executor it selected, making the ARC/non-ARC distinction visible: "Executor: gsd-arc-executor (ARC mode enabled)"

**Phase to address:** ARC-as-Default phase — implement the upgrade detection before flipping the default.

---

## Moderate Pitfalls

---

### Pitfall 5: `npx gsd-code-first@latest` Install Collision with `npx get-shit-done-cc@latest`

**What goes wrong:** Users install both the fork and the upstream. The `install.js` in both writes to the same `.claude/` directory locations. One installation clobbers the other's agent files or config markers. Users experience confusing behavior where commands from one tool call agents from the other.

**Why it happens:** The upstream installer uses file markers (`# GSD Agent Configuration — managed by get-shit-done installer`) without namespace scope. If the fork uses identical markers, both installers claim ownership of the same sections.

**Consequences:**
- Running `npx gsd-code-first@latest` after `npx get-shit-done-cc@latest` silently overwrites upstream agents
- Re-running upstream install after fork install removes fork-specific agents
- Users cannot safely have both installed

**Warning signs:**
- `install.js` in the fork uses the same `GSD_CODEX_MARKER` constant as upstream
- No namespacing of agent files (both write to `.claude/agents/gsd-executor.md`)
- Users report that re-running the upstream installer removes fork agents

**Prevention:**
- Namespace all fork-specific marker constants: `GSD_CF_CODEX_MARKER` (CF = Code-First)
- Fork-specific agents use a distinct prefix or subdirectory that the upstream installer never touches
- Installation docs must warn: "This fork replaces the upstream installation" — do not support concurrent installation
- In `install.js`, detect upstream installation and warn rather than silently overwriting

**Phase to address:** Phase 1 (installer update).

---

### Pitfall 6: The `iterate` Command — Approval Step Blocking in Headless/Scripted Contexts

**What goes wrong:** The `iterate` command is designed as `extract-tags → code-planner → approval → executor`. The approval step requires human input. When users run this in CI, scripts, or other headless contexts, it hangs indefinitely.

**Why it happens:** Interactive approval flows work in the terminal but are not designed for automation.

**Prevention:**
- Follow the upstream's headless prompt overhaul pattern
- Add `--non-interactive` flag to `iterate`
- Always provide a `--yes` auto-approve flag for scripted use cases

**Phase to address:** Phase 2 (iterate command implementation).

---

### Pitfall 7: Context Window Saturation in the `iterate` Workflow

**What goes wrong:** `CODE-INVENTORY.md` for large codebases consumes most of the agent's context window before planning begins.

**Prevention:**
- Support `--phase`/`--scope` filter in `extract-plan`
- Design inventory format to be chunked and sliceable by the planner

**Phase to address:** Phase 2 (extract-plan command design).

---

### Pitfall 9: `set-mode` Config Drift — Phase Mode State is Invisible

**What goes wrong:** The `set-mode` command sets per-phase mode configuration in a config file. Over time developers run `set-mode` multiple times across sessions and forget what modes are active. When the `iterate` command behaves unexpectedly (because it's in `deep-plan` mode), there is no visible reminder.

**Prevention:**
- The `iterate` and `prototype` commands should print the active mode at the start of execution: "Running in code-first mode for phase: implement"
- The `help` command should show the current mode configuration alongside command descriptions

**Phase to address:** Phase 2 (set-mode command implementation).

---

### Pitfall 10: `package.json` Name Collision and npx Caching

**What goes wrong:** When a user has previously run `npx get-shit-done-cc@latest`, npx may cache the old package. After switching to `npx gsd-code-first@latest`, users may unknowingly be running the cached upstream version if npx cache is stale.

**Prevention:**
- Installation docs must include `npx clear-npx-cache` or equivalent
- The installer should print its own name and version prominently at start: "GSD Code-First v1.x.x"

**Phase to address:** Phase 1 (distribution setup).

---

### Pitfall 11: ARC Tags in Generated Code Confuse Future Scanners

**What goes wrong:** The `gsd-prototyper` agent generates code with ARC tags embedded. If that generated code is later passed through the scanner, the scanner may double-count tags.

**Prevention:**
- The scanner should support a `.gsdignore` file to exclude generated code directories
- Default exclusions: `node_modules/`, `dist/`, `build/`, `.planning/`

**Phase to address:** Phase 1 (scanner implementation).

---

### Pitfall 18: Test Agent Running Against Scaffold Stubs — Tests Pass for the Wrong Reason

**What goes wrong:** The prototype pipeline produces scaffold code with stub implementations (`return null`, `throw new Error("not implemented")`). If the test agent runs tests against the scaffold before implementation, some tests will pass against stubs — specifically tests that test the function signature (it exists, it can be called) rather than behavior. These stub-passing tests are committed as green and never re-run after real implementation fills in the stubs. The green signal is stale and misleading.

**Why it happens:** The test agent is given scaffold code and told to write tests. It cannot distinguish "I should write tests that drive implementation" (TDD intent) from "I should write tests that verify current behavior" (validation intent). Without explicit sequencing — test first, then implementation, then verify tests fail against stubs — the agent defaults to validation, which trivially passes against stubs.

**Prevention:**
- The test agent must operate in one of two explicitly declared modes: `draft` (write tests that will fail against current stubs, establishing intent) or `verify` (run tests against completed implementation and report results)
- The review pipeline calls test agent in `verify` mode only — never `draft` mode
- `PROTOTYPE-LOG.md` must record which stubs remain unimplemented so the test runner knows which failures are expected vs. bugs

**Phase to address:** Test-Agent phase — define the two modes before writing the test agent prompt.

---

## Minor Pitfalls

---

### Pitfall 19: PRD Template Overreach — Pipeline Only Works With "Perfect" PRDs

**What goes wrong:** The PRD template designed for the pipeline is so specific that only PRDs written by experienced users with knowledge of the template produce good prototypes. Users with informal PRDs (Notion notes, Slack threads, bullet lists) get poor results and conclude the tool doesn't work for them.

**Prevention:**
- Ship a PRD normalizer step that runs before prototype generation: takes any input format and produces the canonical template structure, with explicit gaps filled in as "TBD" rather than silently dropped
- The normalizer output should always be reviewed by the user before prototype generation begins

**Phase to address:** PRD-to-Prototype Pipeline phase.

---

### Pitfall 20: Review Agent Output Overwriting Previous Review Artifacts

**What goes wrong:** The new review command writes output to a predictable path (e.g., `REVIEW.md` or `REVIEWS.md`). The existing `/gsd:review` command already writes to `{phase_dir}/{padded_phase}-REVIEWS.md`. If the new review command writes to the same path, it overwrites cross-AI plan reviews that may have been used during planning.

**Prevention:**
- New review command must write to a distinct path: `{phase_dir}/{padded_phase}-CODE-REVIEW.md` (not `-REVIEWS.md`)
- Naming audit: before implementing any new command, grep for all existing output file names to identify collision risk

**Phase to address:** Review-Agent phase — name output files before writing the command.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Single agent pass for PRD parsing + scaffold generation | Fewer round-trips | Silent requirement loss when PRD format varies; no user confirmation of what was parsed | Never — always separate parse from generate |
| Test agent in validation mode against scaffold stubs | Tests appear green quickly | False confidence; stub-passing tests never get updated | Never in the review pipeline |
| Naming new review command `/gsd:review` (same as existing) | Simpler for users to remember | Silent command collision; existing cross-AI review breaks | Never |
| Setting `arc.enabled: true` globally on upgrade | All users get ARC by default | Breaks existing non-ARC workflows without warning | Only for new installs |
| Review agent output as single monolithic report | Simple implementation | Verbosity leads to ignoring; no filtering of noise from signal | Only for internal debugging, not user-facing |
| No iteration cap on prototype loop | Agent can keep trying | Unbounded cost and time; divergence compounds | Never in production pipeline |

---

## Integration Gotchas

Common mistakes when connecting new v1.1 commands to existing infrastructure.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| `/gsd:review` (existing) | New review command reuses same command name | Name new command `/gsd:review-code` or `/gsd:verify-prototype`; never shadow existing command |
| `{phase}-REVIEWS.md` (existing plan review artifact) | New review writes to same filename | New review writes to `{phase}-CODE-REVIEW.md` exclusively |
| `gsd-arc-executor` routing in `iterate` | Setting `arc.enabled: true` globally breaks non-ARC users on upgrade | New installs only; detect existing config and preserve on upgrade |
| `PROTOTYPE-LOG.md` | Prototype loop overwrites log on each iteration | Append per-iteration entries with timestamps; never overwrite |
| Test agent and existing `add-tests` command | Two test generation paths with different contracts | `add-tests` is for completed phases post-execution; new test agent is for prototype verification pre-execution — document the distinction explicitly |
| Review agent and `verify-work` command | New review duplicates verification logic | Review agent checks code quality and test results; `verify-work` checks acceptance criteria — complementary, not redundant |

---

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **PRD-to-Prototype Pipeline:** Often missing the requirements confirmation gate — verify that the user sees and approves the parsed requirement list before any code is generated
- [ ] **Test Agent:** Often missing the RED-GREEN discipline — verify that generated tests actually fail against scaffold stubs before implementation fills them in
- [ ] **Review Agent:** Often missing the Judge/filter step — verify that review output has been filtered for actionability before presenting to the user (not raw LLM output)
- [ ] **ARC-as-Default:** Often missing upgrade path handling — verify that existing users with `arc.enabled: false` get a migration notice, not silent behavior change
- [ ] **New Review Command:** Often missing name collision check — verify that the command file is NOT at `.claude/commands/gsd/review.md` and does NOT write to `{phase}-REVIEWS.md`
- [ ] **Autonomous Loop:** Often missing iteration cap — verify that the prototype loop has a hard maximum iteration count defined in config and enforced in code
- [ ] **Test Agent Modes:** Often missing explicit mode declaration — verify that test agent is called with `draft` or `verify` mode, never ambiguously

---

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Autonomous loop diverges | MEDIUM | Kill the loop, read `PROTOTYPE-LOG.md` for last good state, restart prototype from scratch with tighter acceptance criteria |
| Test suite produces false greens | HIGH | Delete generated tests, restart test agent in `draft` mode against stubs, implement functions to make tests pass |
| Review agent output ignored by team | MEDIUM | Disable the agent temporarily, audit past reviews to find which feedback types were used, rewrite judge agent criteria to surface only those types |
| PRD parser misses requirements | MEDIUM | Run PRD normalizer step manually, compare normalized output against original PRD, add missing requirements to the scaffold plan |
| `/gsd:review` collision causes wrong review type | LOW | Rename new command immediately, update any documentation referencing the old name |
| `arc.enabled: true` breaks non-ARC users | MEDIUM | Add `arc.enabled: false` to user config manually, document the config key in the upgrade guide |

---

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Autonomous loop divergence (12) | PRD-to-Prototype Pipeline | Confirm iteration cap is defined in config schema and enforced in prototype command |
| Test false confidence (13, 18) | Test-Agent | Confirm test agent has RED-GREEN gate; run tests against stub implementation and verify they fail |
| Verbose/shallow review (14) | Review-Agent | Confirm two-stage judge architecture is in review agent prompt; confirm output format cap |
| `/gsd:review` collision (15) | Review-Agent | Confirm new command file is at a distinct path; confirm help output shows both commands distinctly |
| PRD parsing brittleness (16) | PRD-to-Prototype Pipeline | Run parser against 3 different PRD formats; confirm requirements confirmation gate exists |
| ARC-as-default upgrade breakage (17) | ARC-as-Default | Confirm installer detects existing `arc.enabled: false` config and does not overwrite it |
| Test agent stub confusion (18) | Test-Agent | Confirm `draft` vs `verify` mode is declared; verify tests fail against unimplemented stubs |
| Review artifact collision (20) | Review-Agent | Grep for `-REVIEWS.md` in new command output paths; confirm it uses `-CODE-REVIEW.md` |

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| ARC standard definition | Standard churn after developer annotates real code | Run one real annotation session before freezing the spec (Pitfall 3) |
| Tag scanner implementation | Regex false positives from strings/URLs | Build test fixture with edge cases at the same time as the scanner (Pitfall 2) |
| Installer / package.json | Install collision with upstream GSD markers | Namespace all marker constants with `GSD_CF_` prefix (Pitfall 5) |
| gsd-executor modification | Fork divergence from upstream executor | Implement as new `gsd-arc-executor.md`, not a patch to existing file (Pitfall 1) |
| extract-plan command | Inventory too large for context window | Design for scope-filtering from the start (Pitfall 7) |
| iterate command | Approval step hangs in CI | Follow upstream headless pattern, add `--non-interactive` flag (Pitfall 6) |
| gsd-code-planner agent | Prompt drift from scanner output format | Define `CODE-INVENTORY.md` schema before writing either component (Pitfall 4) |
| gsd-prototyper agent | Developers skip annotations | Prototyper is the forcing function — must be built before `annotate` (Pitfall 8) |
| PRD-to-prototype loop | Loop diverges, never converges | Hard cap iterations; define acceptance criteria before loop starts (Pitfall 12) |
| PRD-to-prototype loop | PRD format variations cause silent requirement loss | Parse and confirm requirements list before generating scaffold (Pitfall 16) |
| Test-Agent | Tests always pass — false confidence | RED-GREEN gate; test against stubs first; draft vs verify modes (Pitfall 13, 18) |
| Review-Agent | Verbose output gets ignored | Two-stage judge architecture; output cap at 5 items (Pitfall 14) |
| Review-Agent | Command name collision with existing `/gsd:review` | Name new command distinctly; do not place at `review.md` path (Pitfall 15) |
| Review-Agent | Output file overwrites existing plan reviews | Write to `-CODE-REVIEW.md` not `-REVIEWS.md` (Pitfall 20) |
| ARC-as-Default | Breaks existing non-ARC users on upgrade | Detect existing config; print migration notice; only flip default for new installs (Pitfall 17) |

---

## Sources

**v1.0 Pitfalls (Pitfalls 1–11):**
- [Towards Better Comprehension of Breaking Changes in the NPM Ecosystem (FSE 2025)](https://arxiv.org/html/2408.14431v2)
- [Finding Comments in Source Code Using Regular Expressions](https://blog.ostermiller.org/finding-comments-in-source-code-using-regular-expressions/)
- [Prompt Drift: The Hidden Failure Mode Undermining Agentic Systems](https://www.comet.com/site/blog/prompt-drift/)
- [Writing CLI Tools That AI Agents Actually Want to Use](https://dev.to/uenyioha/writing-cli-tools-that-ai-agents-actually-want-to-use-39no)
- [How to Build Multi Agent AI Systems With Context Engineering](https://vellum.ai/blog/multi-agent-systems-building-with-context-engineering)
- [Platform Engineering Maintenance Pitfalls](https://www.cncf.io/blog/2026/01/21/platform-engineering-maintenance-pitfalls-and-smart-strategies-to-stay-ahead/)
- [5 Case Studies on Developer Tool Adoption](https://business.daily.dev/resources/5-case-studies-on-developer-tool-adoption/)

**v1.1 Pitfalls (Pitfalls 12–20):**
- [Stop AI Agent Loops in Autonomous Coding Tasks](https://markaicode.com/fix-ai-agent-looping-autonomous-coding/) — loop divergence root causes and action history tracking (HIGH confidence, 2025)
- [How to Prevent Infinite Loops and Spiraling Costs in Autonomous Agent Deployments](https://codieshub.com/for-ai/prevent-agent-loops-costs) — bounded execution patterns and monitoring (HIGH confidence)
- [Infinite Agent Loop: when an AI agent does not stop](https://www.agentpatterns.tech/en/failures/infinite-loop) — failure taxonomy (MEDIUM confidence)
- [Automated Code Review: The 6-Month Evolution](https://product.hubspot.com/blog/automated-code-review-the-6-month-evolution) — Judge Agent architecture for filtering verbose review output (HIGH confidence, production case study)
- [State of AI Code Review Tools in 2025](https://www.devtoolsacademy.com/blog/state-of-ai-code-review-tools-2025/) — verbosity and signal-to-noise as primary adoption failure mode (MEDIUM confidence)
- [How to write a good spec for AI agents](https://addyosmani.com/blog/good-spec/) — PRD format sensitivity and vagueness as primary agent failure mode (HIGH confidence)
- [How to write PRDs for AI Coding Agents](https://medium.com/@haberlah/how-to-write-prds-for-ai-coding-agents-d60d72efb797) — structural vs. semantic PRD parsing tradeoffs (MEDIUM confidence)
- [AI Chatbots Remain Overconfident — Even When They're Wrong](https://www.cmu.edu/dietrich/news/news-stories/2025/trent-cash-ai-overconfidence) — LLM metacognition weakness underlying test false confidence (HIGH confidence, CMU research 2025)
- Codebase audit: `.claude/commands/gsd/review.md` and `.claude/get-shit-done/workflows/review.md` — existing `/gsd:review` command architecture confirmed (direct inspection)

---
*Pitfalls research for: CLI framework fork with autonomous prototyping pipeline, test generation agents, and review agents*
*Researched: 2026-03-29*
