# Pitfalls Research

**Domain:** CLI framework fork with code annotation system and agent orchestration
**Researched:** 2026-03-29 (v1.0/v1.1) | Updated 2026-03-30 (v1.2 extension)
**Confidence:** HIGH (fork compatibility, regex parsing, agent integration) | MEDIUM (autonomous loop prevention, test generation quality) | HIGH (v1.2 conversational AI output, file aggregation, architecture-mode patterns)

---

> **Note on scope:** Pitfalls 1-11 were written for v1.0. Pitfalls 12-20 were written for v1.1. Pitfalls 21-32 below (after the v1.2 section marker) are specific to the **v1.2 milestone**: `/gsd:brainstorm` conversational command producing PRDs, Feature Map auto-aggregation (`FEATURES.md`), and Architecture Mode for `/gsd:prototype`. The v1.1 tech debt items (known at milestone start) are also catalogued in the Integration Gotchas section.

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

## v1.2 Critical Pitfalls

Pitfalls specific to the v1.2 milestone: `/gsd:brainstorm` conversational command, Feature Map auto-aggregation, and Architecture Mode for `/gsd:prototype`.

---

### Pitfall 21: Brainstorm Output Schema Drift — PRD Format Incompatible With Prototype Pipeline

**What goes wrong:** The `/gsd:brainstorm` command produces a PRD as its final output. The PRD format brainstorm emits is designed around conversation output and looks different from the PRD format the `/gsd:prototype` pipeline expects. When users run `/gsd:prototype` after `/gsd:brainstorm`, the pipeline's AC extractor silently misses requirements because the section headings, list styles, or AC numbering from brainstorm output don't match what prototype.md's semantic extractor was calibrated on.

**Why it happens:** Two developers write two different commands independently. Brainstorm focuses on conversational fidelity (capturing all ideas discussed). Prototype focuses on structured extraction (parsing a clean AC list). Neither validates the handoff at design time. The contract between them is implicit.

**Consequences:**
- `/gsd:brainstorm` → `/gsd:prototype` produces a prototype with fewer ACs than the brainstorm produced, with no warning
- Users trust brainstorm output as complete PRD input, never knowing the handoff is lossy
- The "complete workflow" promise — brainstorm through prototype — is broken at the seam that matters most

**Warning signs:**
- `PROTOTYPE-LOG.md` lists fewer ACs than the brainstorm session identified features
- Brainstorm PRD has `## Ideas` or `## Discussion Summary` sections instead of `## Acceptance Criteria`
- Running AC count from prototype differs from feature count from brainstorm by more than 20%
- Users manually copy-paste brainstorm output into PRD.md, reformatting by hand before running prototype

**Prevention:**
- Define the PRD output schema ONCE, shared by both commands. Brainstorm writes to this schema; prototype reads from this schema. The schema is the contract.
- The brainstorm command's final step must be: "write `BRAINSTORM.md` (transcript) and `.planning/PRD.md` (structured PRD in canonical prototype-compatible format)." Two separate outputs — never just a transcript.
- Add a structural validation step: before closing brainstorm, run the same AC extractor logic that prototype.md uses against the draft PRD and show the user "I found N acceptance criteria. Review and confirm."
- Document the canonical PRD format in a shared reference file (e.g., `get-shit-done/references/prd-format.md`) that both brainstorm and prototype import by reference, not by copy.

**Phase to address:** Brainstorm command phase — define shared PRD schema before writing either the brainstorm agent or updating prototype pipeline.

---

### Pitfall 22: Conversational Brainstorm Losing Structured Data — Ideas Without Traceability

**What goes wrong:** The brainstorm command is conversational. The user discusses features, constraints, and dependencies across many turns. The agent synthesizes these into a PRD at the end. But conversational turns contain nuance that doesn't survive synthesis: a constraint mentioned in turn 3 is absent from the final PRD because it seemed implicit, a dependency identified in turn 7 is merged into a different feature without tracing back, and a rejection decision ("we decided NOT to do X") is silently omitted because the PRD only records what to build, not what was explicitly ruled out.

**Why it happens:** LLM synthesis of long conversations defaults to inclusion of positive decisions (features) and omission of negative decisions (constraints, exclusions, trade-off rationale). The model optimizes for a coherent, forward-looking document rather than a faithful transcript of all decisions. This is the same failure mode that makes meeting notes unreliable.

**Consequences:**
- The prototype implements features the user already decided to defer
- The prototype omits constraints ("must work offline", "must not require authentication") that were stated conversationally but not re-stated in AC form
- Rejected options ("not a mobile app") reappear in the prototype because they weren't explicitly excluded
- Traceability from brainstorm decisions to prototype code is broken — users can't audit why the prototype looks the way it does

**Warning signs:**
- Final PRD has no "Out of Scope" or "Explicitly Deferred" section
- User says during prototype review "we talked about this during brainstorm, why is it included?"
- `BRAINSTORM.md` transcript (turn-by-turn) has more constraints and exclusions than the final `PRD.md`
- The AC list in the PRD contains no negative constraints ("must NOT", "should never")

**Prevention:**
- The brainstorm agent must maintain a running structured ledger alongside the conversation: features confirmed, features deferred, constraints stated, exclusions decided. This is updated after every turn, not just at synthesis time.
- Final synthesis must explicitly populate: `## Acceptance Criteria`, `## Constraints`, `## Out of Scope`, `## Open Questions`. All four sections. The agent must re-read the running ledger before writing each section.
- The user must approve the structured ledger before PRD.md is written: "Here's what I captured — N features, M constraints, K exclusions. Anything missing?"
- Write `BRAINSTORM.md` as a full decision log (not just a transcript), with each decision tagged as `[FEATURE]`, `[CONSTRAINT]`, `[EXCLUDED]`, `[DEFERRED]`, `[OPEN]`.

**Phase to address:** Brainstorm command phase — the structured ledger is a core data structure, not a nice-to-have. Design it before writing the conversational loop.

---

### Pitfall 23: Feature Map Becoming Stale Immediately After Generation

**What goes wrong:** `FEATURES.md` is generated by reading all PRDs and scanning `@gsd-tags` at a point in time. It is a snapshot document. The moment a developer implements a feature (resolving a `@gsd-todo` tag), adds a new tag, or writes a new PRD section, the Feature Map is out of date. Users treat it as authoritative — it is not. The stale map is worse than no map because it actively misleads.

**Why it happens:** Auto-generated dashboard documents are always snapshots. There is no trigger to regenerate them when their source data changes. Unlike `CODE-INVENTORY.md` (regenerated every time `extract-tags` runs), `FEATURES.md` has no natural regeneration trigger in the existing workflow. Without a trigger, it drifts.

**Consequences:**
- Developers make decisions based on Feature Map data that doesn't reflect current code state
- `FEATURES.md` shows a feature as "planned" when it was implemented two iterations ago
- The Feature Map is cited during planning as authoritative, producing plans based on false premises
- Teams stop trusting the Feature Map and revert to manual auditing — defeating the purpose

**Warning signs:**
- `FEATURES.md` shows `@gsd-todo` items that have been resolved in `CODE-INVENTORY.md`
- The `last updated` timestamp in `FEATURES.md` is more than one iteration old
- A developer says "this feature is done" but the Feature Map still shows it as planned
- The implementation status in `FEATURES.md` diverges from `CODE-INVENTORY.md` tag counts

**Prevention:**
- `FEATURES.md` must be regenerated as part of the `extract-tags` workflow — every time tags are extracted, features are re-aggregated. Not a separate on-demand command, a coupled step.
- Add a `last-updated` and `source-hash` to `FEATURES.md` so readers know immediately whether it's current: "Generated from CODE-INVENTORY.md (last extract: 2026-03-30, hash: abc123)"
- The `/gsd:brainstorm` command must trigger a Feature Map regeneration when it writes a new PRD, so the map reflects the new planned features immediately.
- Document explicitly in the Feature Map header: "This file is auto-generated. Do not edit manually. Run `extract-tags` to refresh."
- Treat manual edits to `FEATURES.md` as a bug — if a user edits it, the next `extract-tags` run overwrites their changes silently. Either prohibit manual edits or support a `## User Notes` section that survives regeneration.

**Phase to address:** Feature Map phase — couple regeneration to `extract-tags` at design time. Do not design Feature Map as a stand-alone command.

---

### Pitfall 24: Feature Map Aggregation Producing Contradictory Feature Status

**What goes wrong:** The Feature Map reads from two sources: PRDs (planned features) and `@gsd-tags` in code (implementation status). These sources can contradict each other. A PRD lists "offline support" as a planned feature. `CODE-INVENTORY.md` has no `@gsd-todo(ref:AC-offline)` tag anywhere in the codebase. The Feature Map must decide: is "offline support" planned-but-not-started, or was it dropped? It cannot tell. It picks one, silently.

**Why it happens:** Aggregating across two heterogeneous sources (prose documents and structured annotations) produces ambiguity when the sources don't agree. PRDs describe intent; code describes state. Intent and state can diverge legitimately (feature deferred mid-iteration) or accidentally (tag forgotten). The aggregator has no way to distinguish the two cases.

**Consequences:**
- Feature Map shows "planned" features that were actually deferred — produces false planning pressure
- Feature Map shows "not started" features that have partial implementations (no tag written yet) — produces false gaps
- Developers lose confidence in the Feature Map's ability to report actual project state
- Planning decisions are made on contradictory data

**Warning signs:**
- Feature Map lists features from PRD with no corresponding `@gsd-todo` tags
- Feature Map shows 100% planned features as "not started" for a project that has been iterating for weeks
- Discrepancy between PRD AC count and CODE-INVENTORY.md AC-linked tag count exceeds 30%
- Users manually annotate `FEATURES.md` with status corrections

**Prevention:**
- The Feature Map aggregator must output an explicit "unresolved" category for features present in PRD but absent from code tags: "Planned in PRD, no implementation tags found — confirm status."
- The aggregator must never silently pick a status when sources contradict. It must surface contradictions as a separate section: `## Needs Clarification`.
- Design the brainstorm-to-prototype pipeline so that every PRD AC has exactly one `@gsd-todo(ref:AC-N)` tag. This is already guaranteed by prototype.md's AC extraction — but only if brainstorm output uses the correct PRD format (see Pitfall 21). Feature Map aggregation quality depends entirely on PRD → prototype handoff fidelity.
- When generating Feature Map, compare PRD AC count to CODE-INVENTORY.md AC-linked tag count and report the delta explicitly: "PRD has 12 ACs; CODE-INVENTORY.md has 10 AC-linked tags. 2 ACs may be untracked."

**Phase to address:** Feature Map phase — design the contradiction-handling logic before building the aggregator. A Feature Map that silently picks wrong is worse than no Feature Map.

---

### Pitfall 25: Architecture Mode Skeleton Not Matching Project Conventions — Dead Scaffold

**What goes wrong:** Architecture Mode for `/gsd:prototype` generates a structural skeleton first (directories, interfaces, module boundaries) before any implementation. The skeleton uses conventions from the agent's training data — generic Node.js project layout, conventional file naming, standard interface patterns. The actual project has its own conventions: different naming, different module boundaries, different config patterns. The generated skeleton is technically valid but diverges from the project's actual patterns, creating a codebase that looks "off" and requires significant manual correction before it's usable.

**Why it happens:** Architecture Mode is conceptually independent from the project context — it generates structure, not just code. But structure is deeply convention-driven. Agents default to the most common pattern from training data when not given explicit convention context. The more opinionated the existing codebase, the larger the gap between generated skeleton and project reality.

**Consequences:**
- Developers spend more time correcting skeleton conventions than they saved from the skeleton generation
- The skeleton is abandoned and implementation proceeds without architecture scaffolding, defeating the mode's purpose
- `@gsd-tags` in the skeleton reference architecture decisions that don't apply — CODE-INVENTORY.md is polluted with false context
- Users conclude "architecture mode doesn't understand our project" after one failed attempt

**Warning signs:**
- Generated skeleton uses different naming conventions than existing project files (e.g., `camelCase.js` when project uses `kebab-case.js`)
- Skeleton creates directories that already exist with different internal structure
- Interface names in skeleton don't follow existing patterns visible in the project
- Skeleton generates a module structure incompatible with the project's import patterns (`require` vs `import`, CJS vs ESM)

**Prevention:**
- Architecture Mode must read the project's existing conventions BEFORE generating skeleton. Required inputs: existing file structure (via `ls` or Glob), existing naming patterns (from actual filenames), module system in use (from `package.json` `"type"` field).
- The gsd-prototyper agent in architecture mode must explicitly state in its prompt: "Before generating any files, identify the project's naming conventions, module system, and directory structure. Generate the skeleton using ONLY conventions you observe in the existing project."
- For new projects with no existing code, architecture mode must ask the user for convention choices before generating: "What naming convention? (kebab-case / camelCase / PascalCase)". This is not optional overhead — it is mandatory to avoid an unusable skeleton.
- The skeleton should be shown to the user for approval before any files are written. Architecture decisions are expensive to reverse.

**Phase to address:** Architecture Mode phase — add the convention-reading step to the gsd-prototyper prompt before architecture mode generates any output.

---

### Pitfall 26: Architecture Mode Producing Overly Deep Skeletons — Premature Over-Engineering

**What goes wrong:** Architecture Mode generates a complete structural skeleton with every directory, every interface, every module boundary anticipated. For a project that turns out to need 3 modules, it generates 12. The skeleton prescribes an architecture that front-loads complexity the project doesn't need. Developers feel compelled to fill every generated file, producing bloated codebases. Or they abandon the skeleton and write from scratch, feeling the tool "got in the way."

**Why it happens:** Agents optimizing for "comprehensive architecture" default to enterprise-scale patterns from training data. Generating a skeleton that "anticipates future scale" is praised in code review training data. Minimalism requires explicit instruction; comprehensiveness is the default. This is particularly acute for architecture-mode agents because their entire task is structure generation — without a scope constraint, they generate all the structure they can.

**Consequences:**
- 40% of generated files never get implemented — dead code from day one
- The project looks like it has a complete architecture but most modules are empty stubs
- CODE-INVENTORY.md has `@gsd-todo` tags in skeleton files that will never be resolved, inflating "planned" counts
- Iteration plans include work for skeleton modules that were never meant to be built

**Warning signs:**
- Generated skeleton has more than 2x the number of files the PRD implies
- Skeleton contains modules with names like `future/`, `v2/`, `advanced/`, or `enterprise/`
- The user says "this is more structure than we need" during skeleton review
- A significant fraction of skeleton files have only `@gsd-todo` tags and no code

**Prevention:**
- Architecture Mode must derive scope from the PRD AC count, not from architectural completeness. The prompt constraint is: "Generate the minimum skeleton that can accommodate the N acceptance criteria in the PRD. Do not generate modules or directories not required by these ACs."
- Show skeleton scope to user before generation: "Based on N ACs, I'll generate M modules. Is this scope appropriate?" — user can reduce scope before any files are written.
- Skeleton files should be generated with exactly one `@gsd-todo` per AC they address, and no extra scaffolding files. No "placeholder for future expansion" files.
- Add a "skeleton audit" gate: after generation, count generated files vs. PRD ACs. If files > (ACs * 2), warn: "Skeleton may be over-engineered. Review before proceeding."

**Phase to address:** Architecture Mode phase — add explicit scope-bounding instructions to gsd-prototyper's architecture mode prompt.

---

### Pitfall 27: Brainstorm-to-Prototype Integration — AC Numbering Collision

**What goes wrong:** The brainstorm session produces a PRD with ACs numbered AC-1 through AC-N. The user runs `/gsd:prototype` which extracts ACs, re-numbers them sequentially, and embeds `@gsd-todo(ref:AC-N)` tags. If the user later runs another brainstorm session and appends new ACs, the numbering from the new PRD conflicts with the existing tags in code. `AC-3` in the new PRD is a different requirement than `AC-3` in the existing code tags.

**Why it happens:** AC numbering is local to the extraction session in the current prototype pipeline design. There is no global AC registry. When two brainstorm sessions produce PRDs and both are fed into prototype, the numbering resets from AC-1 each time. The code tags from the first session and the new code tags from the second session share the AC-N namespace but reference different requirements.

**Consequences:**
- `CODE-INVENTORY.md` has two different `@gsd-todo(ref:AC-3)` tags referencing two different requirements
- Review agent's Stage 1 (spec compliance) finds AC-3 "resolved" because a tag exists, but it's the wrong AC
- Feature Map aggregation is corrupted — AC coverage appears complete but is actually mixed
- Users cannot trace a tag in code back to its requirement in the PRD without checking the date

**Warning signs:**
- Multiple brainstorm sessions have produced multiple PRD files
- Same AC number appears in `CODE-INVENTORY.md` with conflicting descriptions
- Review agent Stage 1 passes but user-visible features are missing
- `PRD.md` has been overwritten by a second brainstorm session (replacing AC-1 with a different requirement)

**Prevention:**
- Brainstorm must append to `.planning/PRD.md`, never overwrite it. New ACs from a second session get numbers that continue from the existing highest AC number (AC-13 if the existing PRD has 12 ACs).
- The brainstorm command must read existing `.planning/PRD.md` before starting: "You currently have 12 ACs. This session will produce ACs starting at AC-13."
- Alternatively, use a separate PRD file per brainstorm session (`PRD-v2.md`, `PRD-v3.md`) and require an explicit merge step before running prototype. Never auto-merge.
- The prototype command must validate that the AC numbers it's about to embed don't collide with existing tags: scan CODE-INVENTORY.md for the highest existing AC reference before numbering new ones.

**Phase to address:** Brainstorm command phase AND prototype pipeline integration phase — the AC numbering contract must be defined as a shared constraint, not independently by each command.

---

## Moderate Pitfalls

---

### Pitfall 5: `npx gsd-code-first@latest` Install Collision with `npx get-shit-done-cc@latest`

**What goes wrong:** Users install both the fork and the upstream. The `install.js` in both writes to the same `.claude/` directory locations. One installation clobbers the other's agent files or config markers. Users experience confusing behavior where commands from one tool call agents from the other.

**Why it happens:** The upstream installer uses file markers without namespace scope. If the fork uses identical markers, both installers claim ownership of the same sections.

**Prevention:**
- Namespace all fork-specific marker constants: `GSD_CF_CODEX_MARKER` (CF = Code-First)
- Fork-specific agents use a distinct prefix or subdirectory that the upstream installer never touches
- Installation docs must warn: "This fork replaces the upstream installation" — do not support concurrent installation
- In `install.js`, detect upstream installation and warn rather than silently overwriting

**Phase to address:** Phase 1 (installer update).

---

### Pitfall 6: The `iterate` Command — Approval Step Blocking in Headless/Scripted Contexts

**What goes wrong:** The `iterate` command's approval step requires human input. When users run this in CI, scripts, or other headless contexts, it hangs indefinitely.

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

**What goes wrong:** The `set-mode` command sets per-phase mode configuration in a config file. Over time developers run `set-mode` multiple times across sessions and forget what modes are active.

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

**What goes wrong:** The prototype pipeline produces scaffold code with stub implementations (`return null`, `throw new Error("not implemented")`). If the test agent runs tests against the scaffold before implementation, some tests will pass against stubs — specifically tests that test the function signature (it exists, it can be called) rather than behavior.

**Prevention:**
- The test agent must operate in one of two explicitly declared modes: `draft` (write tests that will fail against current stubs, establishing intent) or `verify` (run tests against completed implementation and report results)
- The review pipeline calls test agent in `verify` mode only — never `draft` mode
- `PROTOTYPE-LOG.md` must record which stubs remain unimplemented

**Phase to address:** Test-Agent phase — define the two modes before writing the test agent prompt.

---

### Pitfall 28: Brainstorm Session Context Window Pressure — Long Conversations Lose Early Decisions

**What goes wrong:** `/gsd:brainstorm` is conversational. Long sessions (15+ turns exploring a complex product) push early decisions out of the active context window. The agent's synthesis at the end of the session reflects only the most recent turns. Constraints stated in turn 2, requirements discussed in turn 5, and trade-offs decided in turn 8 are not in the final PRD because they are no longer in the model's effective attention when synthesis happens.

**Why it happens:** Context windows are finite and attention is not uniform across their length. Long conversations front-load important decisions that get diluted by the weight of later content. This is a known failure mode for multi-turn conversational agents that synthesize at the end. The model does not "forget" early turns, but its synthesis is disproportionately influenced by recent turns.

**Consequences:**
- PRD is biased toward whatever was most recently discussed, not the most important decisions
- Early constraints are missing from PRD.md, causing prototype to ignore them
- Users who had long, thorough brainstorm sessions get worse PRDs than users with short sessions

**Warning signs:**
- PRD.md has more features discussed in turns 10-15 than in turns 1-5, despite equal importance
- Constraints stated early in session don't appear in `## Constraints` section of PRD
- User reading the PRD says "we talked about X in the beginning, why isn't it here?"

**Prevention:**
- Maintain a running decision ledger updated AFTER EVERY TURN (see Pitfall 22) — this keeps early decisions in active context throughout the session regardless of session length
- At the midpoint of a long session (after turn 7-8), pause and summarize: "Here's what we've decided so far. Confirm before continuing." This forces a checkpoint that brings early decisions back into focus.
- The synthesis step must read the decision ledger first, not the conversation transcript. The ledger is the canonical source; the transcript is supplementary.
- Hard cap the brainstorm session at 20 turns. If the session is still open at turn 20, surface a warning: "This session is long. Consider ending now and synthesizing — you can run another brainstorm session for additional scope."

**Phase to address:** Brainstorm command phase — the decision ledger is required infrastructure, not optional. It must be in the agent design from turn 1.

---

### Pitfall 29: Feature Map Scope Creep — Aggregating Everything, Signaling Nothing

**What goes wrong:** The Feature Map aggregator reads all PRDs and all `@gsd-tags` across the entire project. On a project with multiple milestones, this produces a Feature Map with hundreds of entries spanning past, current, and future milestones. The map is technically comprehensive but practically useless — developers cannot identify what is relevant to the current iteration. The signal-to-noise ratio approaches zero.

**Why it happens:** Auto-aggregation without scope filtering defaults to "include everything." This is the safe choice for the aggregator (it doesn't miss anything) but the wrong choice for the user (they can't find anything). The same problem occurs with CODE-INVENTORY.md on large codebases (Pitfall 7) but is worse for Feature Map because features accumulate across milestones.

**Consequences:**
- Feature Map is not consulted during planning because it's too large to scan quickly
- Current milestone's features are buried under completed and future milestone features
- Teams create a separate manual tracking document, defeating the auto-aggregation purpose

**Warning signs:**
- `FEATURES.md` has more than 50 entries on a project that's only in its second milestone
- Features marked `[COMPLETE]` from previous milestones constitute more than 40% of the map
- Planning discussions reference "the feature map" but nobody has opened it in the last week

**Prevention:**
- Feature Map must support milestone scoping: `extract-features --milestone current` shows only the current milestone's features. The full unscoped map is available but is not the default.
- Current milestone features are shown first, with completed and future milestones collapsed to summary counts: "Completed: 18 features (v1.0, v1.1) | Current: 7 features (v1.2) | Planned: 3 features (v1.3+)"
- Include a `## Summary` at the top of FEATURES.md with current milestone stats before the full listing — this makes the map scannable without reading the whole document.

**Phase to address:** Feature Map phase — build milestone scoping into the aggregator from the start. Do not build a flat aggregator and add scoping later.

---

## Minor Pitfalls

---

### Pitfall 19: PRD Template Overreach — Pipeline Only Works With "Perfect" PRDs

**What goes wrong:** The PRD template designed for the pipeline is so specific that only PRDs written by experienced users produce good prototypes. Users with informal PRDs get poor results and conclude the tool doesn't work for them.

**Prevention:**
- Ship a PRD normalizer step that runs before prototype generation: takes any input format and produces the canonical template structure, with explicit gaps filled in as "TBD" rather than silently dropped
- The normalizer output should always be reviewed by the user before prototype generation begins

**Phase to address:** PRD-to-Prototype Pipeline phase.

---

### Pitfall 20: Review Agent Output Overwriting Previous Review Artifacts

**What goes wrong:** The new review command writes output to a predictable path. The existing `/gsd:review` command already writes to `{phase_dir}/{padded_phase}-REVIEWS.md`. If the new review command writes to the same path, it overwrites cross-AI plan reviews.

**Prevention:**
- New review command must write to a distinct path: `{phase_dir}/{padded_phase}-CODE-REVIEW.md` (not `-REVIEWS.md`)
- Naming audit: before implementing any new command, grep for all existing output file names to identify collision risk

**Phase to address:** Review-Agent phase — name output files before writing the command.

---

### Pitfall 30: Architecture Mode Creating Files Outside Planning Layer — Polluting User Codebase

**What goes wrong:** Architecture Mode generates skeleton files as part of a prototype run. If the output directory is not scoped correctly, skeleton files land in the user's project root or in directories that already contain production code. Unlike implementation code (which can be reviewed and merged), skeleton files with stub implementations silently co-exist with real code, producing duplicate symbols, import conflicts, or test failures.

**Why it happens:** The existing `/gsd:prototype` command writes to a configurable target directory that defaults to the project root. Architecture Mode, which is an enhancement to prototype, inherits this default. Skeleton generation at the project root on a brownfield codebase contaminates existing code.

**Prevention:**
- Architecture Mode must default to `.planning/prototype/skeleton/` not the project root
- Warn explicitly if the target directory contains non-empty subdirectories: "Target directory contains existing code. Architecture Mode will add skeleton files alongside existing code. Confirm? [yes / specify different directory]"
- Skeleton files must have a distinct marker comment: `// @gsd-skeleton — generated by architecture mode` to distinguish them from implementation code during review

**Phase to address:** Architecture Mode phase — enforce skeleton output directory before writing the command.

---

### Pitfall 31: V1.1 Tech Debt Propagation — Known Stale References Causing v1.2 Defects

**What goes wrong:** Two known tech debt items from v1.1 are documented in `PROJECT.md`: (1) `gsd-tester.md:221` references `extract-plan` instead of `extract-tags` — the stale command name; (2) `review-code.md:103` uses `grep -oP` which fails on macOS BSD grep (only GNU grep supports `-P` PCRE flag). If v1.2 work copies patterns from either of these files without auditing them, the stale patterns propagate. A brainstorm command that looks at review-code.md for pattern inspiration will copy the non-portable grep. A feature map generator that looks at gsd-tester.md for tag-scanning patterns will reference `extract-plan`.

**Why it happens:** Developers use existing code as templates. When existing code has bugs, the bugs propagate to new code written from those templates. The tech debt items are documented in PROJECT.md but developers don't necessarily read PROJECT.md before looking at example files.

**Consequences:**
- New v1.2 commands fail on macOS with `grep: invalid option -- P` (same as review-code.md:103)
- Feature Map generator references `extract-plan` instead of `extract-tags`, producing "command not found" errors
- Tech debt accumulates as new files inherit bugs from old files

**Warning signs:**
- Any new command file contains `grep -oP` or `grep -P`
- Any new agent or command references `extract-plan` as a tool to call
- A new file was "modeled on" `review-code.md` or `gsd-tester.md` without auditing the source

**Prevention:**
- Fix both tech debt items BEFORE v1.2 development begins, not alongside it. The v1.2 brainstorm phase should include a "fix known debt" task as its first step.
- Fix for `gsd-tester.md:221`: replace `extract-plan` with `extract-tags`
- Fix for `review-code.md:103`: replace `grep -oP "ref:AC-\d+"` with `grep -o "ref:AC-[0-9]*"` (portable POSIX grep)
- Add a CI lint rule: scan all command/agent files for `grep -oP` or `grep -P` and fail if found
- When writing new command files, explicitly state the source model (if any) and run a diff against it to identify inherited patterns that need auditing

**Phase to address:** First task in v1.2 Brainstorm phase — resolve both debt items before writing any new files.

---

### Pitfall 32: Feature Map and CODE-INVENTORY.md Circular Dependency — Which Is Source of Truth?

**What goes wrong:** `FEATURES.md` is generated by aggregating from PRDs and `CODE-INVENTORY.md`. `CODE-INVENTORY.md` is generated by scanning `@gsd-tags`. Both documents describe "what features exist and in what state." When they diverge, there is no defined source of truth. Developers update `FEATURES.md` manually to correct a status, then `extract-tags` regenerates `CODE-INVENTORY.md` without consulting `FEATURES.md`, and the two diverge again. Or they update `@gsd-tags` to match `FEATURES.md`, then regenerate `FEATURES.md` from code, and the manual corrections are lost.

**Why it happens:** Two auto-generated documents describing the same domain creates an implicit circular dependency without clear ownership. Which document drives the other? In the current design, `CODE-INVENTORY.md` → `FEATURES.md` (features are aggregated from code). But users naturally want to edit `FEATURES.md` (it's the higher-level view) and have code reflect it. This is the wrong direction for an annotation-first system.

**Prevention:**
- Establish an explicit and documented single direction: `@gsd-tags in code` → `CODE-INVENTORY.md` → `FEATURES.md`. Tags are the source of truth. `FEATURES.md` is always derived, never edited.
- `FEATURES.md` must include a read-only notice in its header that is regenerated on every run: "AUTO-GENERATED — do not edit manually. Source of truth: @gsd-tags in codebase. Last regenerated: {timestamp}."
- If a user needs to change a feature's status, they must update the `@gsd-tag` in the code, not `FEATURES.md`. Document this workflow explicitly.
- If the Feature Map needs user-maintained content (e.g., roadmap decisions, priority notes), provide a separate `FEATURES-NOTES.md` that is NOT auto-generated and is merged into the display but not into the auto-generated source.

**Phase to address:** Feature Map phase — establish the single source of truth direction before building the aggregator.

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
| Brainstorm synthesizing at session end without running ledger | Simpler implementation | Early decisions lost in long sessions; PRD biased to recent turns | Never for sessions over 10 turns |
| Feature Map as on-demand command, not coupled to extract-tags | Less coupling | Map goes stale immediately after any code change | Only for one-time audit reports, not ongoing tracking |
| Architecture Mode using training-data defaults for project conventions | No upfront convention elicitation | Skeleton diverges from project patterns; requires manual correction | Never for brownfield projects |
| Copying patterns from gsd-tester.md or review-code.md without auditing | Faster development | Propagates known tech debt (grep -oP, extract-plan stale refs) into new files | Never — always audit source files before using as templates |

---

## Integration Gotchas

Common mistakes when connecting new v1.2 commands to existing infrastructure.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| `/gsd:review` (existing) | New review command reuses same command name | Name new command `/gsd:review-code` or `/gsd:verify-prototype`; never shadow existing command |
| `{phase}-REVIEWS.md` (existing plan review artifact) | New review writes to same filename | New review writes to `{phase}-CODE-REVIEW.md` exclusively |
| `gsd-arc-executor` routing in `iterate` | Setting `arc.enabled: true` globally breaks non-ARC users on upgrade | New installs only; detect existing config and preserve on upgrade |
| `PROTOTYPE-LOG.md` | Prototype loop overwrites log on each iteration | Append per-iteration entries with timestamps; never overwrite |
| Test agent and existing `add-tests` command | Two test generation paths with different contracts | `add-tests` is for completed phases post-execution; new test agent is for prototype verification pre-execution — document the distinction explicitly |
| Review agent and `verify-work` command | New review duplicates verification logic | Review agent checks code quality and test results; `verify-work` checks acceptance criteria — complementary, not redundant |
| `/gsd:brainstorm` output → `/gsd:prototype` input | Brainstorm PRD format incompatible with prototype AC extractor | Define shared canonical PRD schema; brainstorm writes to it, prototype reads from it (Pitfall 21) |
| Feature Map regeneration | Running `/gsd:brainstorm` without regenerating Feature Map | Brainstorm must trigger feature map regeneration as final step when PRD.md is written |
| Feature Map + CODE-INVENTORY.md | Manual edits to FEATURES.md overwritten by next extract-tags | FEATURES.md is auto-generated only; user notes go in FEATURES-NOTES.md (Pitfall 32) |
| Architecture Mode + existing project files | Skeleton files landing in project root alongside production code | Architecture Mode outputs to `.planning/prototype/skeleton/` by default; warns if target directory is non-empty |
| `gsd-tester.md:221` stale ref | New feature map or brainstorm agent copies extract-plan from tester | Fix stale ref in gsd-tester.md before writing new files; use `extract-tags` not `extract-plan` |
| `review-code.md:103` non-portable grep | New commands modeling on review-code.md copy `grep -oP` | Fix non-portable grep in review-code.md first; use `grep -o "ref:AC-[0-9]*"` instead |
| AC numbering across multiple brainstorm sessions | Second brainstorm session starts AC numbering from AC-1 | Brainstorm reads existing PRD.md and continues from highest existing AC number (Pitfall 27) |

---

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Feature Map scanning all PRDs + all source files on every regeneration | Slow `extract-tags` on large projects | Incremental scan: only re-read files changed since last scan (hash-based) | Projects with 50+ source files and frequent iteration |
| Brainstorm session with no turn limit produces 30-turn conversations | Synthesis step reads huge context; early decisions lost | Hard cap at 20 turns with explicit mid-session summary | Sessions with 3+ users or complex product exploration |
| Architecture Mode generating skeletons with 50+ stub files | Skeleton takes multiple minutes to generate; user waits | Scope skeleton to PRD AC count; warn if skeleton exceeds ACs × 2 | PRDs with more than 15 ACs in a single architecture mode run |
| `FEATURES.md` loaded in full as context for every brainstorm turn | Context window consumed by stale feature data | Load only current-milestone features as brainstorm context | Projects beyond v1.2 with 100+ total features |

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
- [ ] **Brainstorm command:** Often missing structured decision ledger — verify the agent maintains a running `[FEATURE] / [CONSTRAINT] / [EXCLUDED] / [DEFERRED]` ledger that is updated after every turn, not just at synthesis
- [ ] **Brainstorm command:** Often missing AC continuity — verify brainstorm reads existing PRD.md before numbering new ACs, continuing from the highest existing number (not resetting to AC-1)
- [ ] **Feature Map:** Often missing staleness indicator — verify `FEATURES.md` includes a `last-updated` timestamp and is regenerated as part of `extract-tags`, not independently
- [ ] **Feature Map:** Often missing contradiction handling — verify the aggregator has an explicit `## Needs Clarification` section for features present in PRD but absent from code tags
- [ ] **Architecture Mode:** Often missing convention reading step — verify gsd-prototyper in architecture mode reads existing project conventions (file naming, module system) before generating any skeleton files
- [ ] **Architecture Mode:** Often missing scope bound — verify skeleton file count does not exceed PRD AC count × 2 without user confirmation
- [ ] **V1.1 tech debt:** Often missing fix before new work begins — verify `gsd-tester.md:221` uses `extract-tags` not `extract-plan`, and `review-code.md:103` uses portable `grep -o` not `grep -oP`

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
| Brainstorm PRD format incompatible with prototype | MEDIUM | Manually edit PRD.md to match canonical format (add `## Acceptance Criteria` section, renumber ACs sequentially); run prototype again |
| Feature Map becomes stale and misleading | LOW | Run `extract-tags` to regenerate; if manual edits were made, they are lost — accept the loss, do not edit FEATURES.md manually going forward |
| Architecture skeleton diverges from project conventions | HIGH | Delete generated skeleton files, re-run Architecture Mode with explicit convention instructions passed as arguments, confirm skeleton before files are written |
| AC numbering collision from two brainstorm sessions | MEDIUM | Manually renumber ACs in newer PRD sections and all corresponding `@gsd-todo` tags in code; re-run `extract-tags` to refresh CODE-INVENTORY.md |
| V1.1 tech debt propagated into new files | LOW | Grep for `grep -oP` and `extract-plan` across new files; fix each occurrence; add lint rule to prevent recurrence |

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
| Brainstorm PRD schema drift (21) | Brainstorm command — Phase 1: Define shared PRD schema | Verify brainstorm output passes prototype AC extractor without manual correction |
| Brainstorm decision loss (22) | Brainstorm command — Agent design | Verify decision ledger has all 4 categories and is updated per turn; verify PRD has Out of Scope section |
| Feature Map staleness (23) | Feature Map phase — Couple to extract-tags | Verify FEATURES.md timestamp updates every time extract-tags runs |
| Feature Map contradictions (24) | Feature Map phase — Aggregator logic | Verify aggregator outputs `## Needs Clarification` section when PRD AC count diverges from code tag count |
| Architecture skeleton convention mismatch (25) | Architecture Mode phase — Convention reading | Verify gsd-prototyper reads package.json and existing file structure before skeleton generation |
| Architecture skeleton over-engineering (26) | Architecture Mode phase — Scope bounding | Verify skeleton file count ≤ (AC count × 2); confirm scope shown to user before generation |
| AC numbering collision (27) | Brainstorm command AND prototype pipeline | Verify brainstorm reads existing PRD.md before numbering; verify prototype detects existing tag numbers |
| Brainstorm context window pressure (28) | Brainstorm command — turn structure | Verify turn cap at 20; verify mid-session summary trigger exists |
| Feature Map scope creep (29) | Feature Map phase — Milestone scoping | Verify default view is current milestone only; full view requires explicit flag |
| Skeleton file contamination (30) | Architecture Mode phase — Output directory | Verify default output is `.planning/prototype/skeleton/`; verify warning on non-empty target |
| V1.1 tech debt propagation (31) | First task in Brainstorm phase | Verify gsd-tester.md:221 fixed; verify review-code.md:103 fixed; lint rule added |
| Feature Map circular dependency (32) | Feature Map phase — Source of truth docs | Verify FEATURES.md has read-only header; verify no workflow writes to FEATURES.md except extract-tags |

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
| Brainstorm command design | PRD output incompatible with prototype AC extractor | Define shared PRD schema before writing either command (Pitfall 21) |
| Brainstorm command design | Early session decisions lost in synthesis | Build per-turn decision ledger from turn 1, not as afterthought (Pitfall 22, 28) |
| Brainstorm command design | AC numbering collision across sessions | Brainstorm reads existing PRD.md before numbering new ACs (Pitfall 27) |
| Brainstorm command | V1.1 tech debt files used as templates | Fix gsd-tester.md:221 and review-code.md:103 before writing any new files (Pitfall 31) |
| Feature Map aggregator | Map goes stale between iterations | Couple Feature Map regeneration to extract-tags workflow (Pitfall 23) |
| Feature Map aggregator | PRD-vs-code contradictions picked silently | Aggregator must output `## Needs Clarification` for unresolved features (Pitfall 24) |
| Feature Map aggregator | Too many features to scan; scope creep | Default to current milestone; full view requires `--all-milestones` flag (Pitfall 29) |
| Feature Map + CODE-INVENTORY.md | Two documents, no clear source of truth | Tags in code are source of truth; both documents are derived (Pitfall 32) |
| Architecture Mode prototype | Skeleton uses wrong project conventions | Read package.json, existing file names, and module patterns before generating (Pitfall 25) |
| Architecture Mode prototype | Skeleton over-engineers structure | Scope to PRD ACs; cap at AC count × 2 files; confirm scope with user (Pitfall 26) |
| Architecture Mode prototype | Skeleton contaminates existing codebase | Default output to `.planning/prototype/skeleton/`; warn on non-empty target (Pitfall 30) |

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

**v1.2 Pitfalls (Pitfalls 21–32):**
- Codebase audit: `commands/gsd/prototype.md` — AC extraction logic, iteration loop design, and PRD format expectations confirmed (HIGH confidence, direct read)
- Codebase audit: `commands/gsd/review-code.md` lines 97–103 — `grep -oP` portability defect confirmed on macOS BSD grep (HIGH confidence, direct inspection)
- Codebase audit: `agents/gsd-tester.md` line 221 — stale `extract-plan` reference confirmed (HIGH confidence, direct inspection)
- `.planning/PROJECT.md` — v1.1 tech debt items documented as known issues (HIGH confidence)
- [Conversation summarization failures in LLM agents — context recency bias](https://arxiv.org/abs/2307.03987) — LLM attention non-uniformity in long conversations; recency bias in synthesis (MEDIUM confidence, academic source)
- [Building reliable multi-step AI pipelines — contract testing between agent outputs](https://eugeneyan.com/writing/llm-patterns/) — input/output contract validation between pipeline stages (HIGH confidence, Eugene Yan, widely cited)
- [Dashboard document freshness in software projects](https://martinfowler.com/articles/agileFluency.html) — stale generated documents as planning liability (MEDIUM confidence, Fowler's commentary on living documents)
- Pattern recognition from v1.1 defects: command name collision (`review` vs `review-code`), non-portable shell commands (`grep -oP`) — these two defects from v1.1 directly inform Pitfall 31 (propagation risk) and integration gotchas table (HIGH confidence, direct post-mortem)

---
*Pitfalls research for: CLI framework fork with autonomous prototyping pipeline, conversational brainstorm command, feature map auto-aggregation, and architecture-mode prototyping*
*Researched: 2026-03-29 (v1.0/v1.1) | Updated: 2026-03-30 (v1.2 extension)*
