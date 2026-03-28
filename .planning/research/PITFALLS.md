# Domain Pitfalls

**Domain:** CLI framework fork with code annotation system and agent orchestration
**Researched:** 2026-03-28
**Confidence:** HIGH (fork compatibility, regex parsing) | MEDIUM (agent orchestration, adoption)

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

## Minor Pitfalls

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
- Package name `gsd-code-first` is sufficiently distinct to avoid npx cache collision in practice — this is LOW risk but worth documenting

**Phase to address:** Phase 1 (distribution setup).

---

### Pitfall 11: ARC Tags in Generated Code Confuse Future Scanners

**What goes wrong:** The `gsd-prototyper` agent generates code with ARC tags embedded. If that generated code is later passed through the scanner, the scanner may double-count tags (e.g., a tag in a test fixture or a template string that contains tag syntax).

**Prevention:**
- The scanner should support a `.gsdignore` file (similar to `.gitignore`) to exclude generated code directories
- Default exclusions: `node_modules/`, `dist/`, `build/`, `.planning/`

**Phase to address:** Phase 1 (scanner implementation).

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
| gsd-prototyper agent | Developers skip annotations, use retroactive annotator instead | Prototyper is the forcing function — must be built before `annotate` (Pitfall 8) |
| set-mode command | Active mode is invisible, causes confusion | Print active mode at command start (Pitfall 9) |

---

## Sources

- [Towards Better Comprehension of Breaking Changes in the NPM Ecosystem (FSE 2025)](https://arxiv.org/html/2408.14431v2) — breaking change propagation in npm forks
- [Finding Comments in Source Code Using Regular Expressions](https://blog.ostermiller.org/finding-comments-in-source-code-using-regular-expressions/) — regex comment parsing edge cases
- [Avoiding Edge Cases in Node.js: Regex Challenges Unveiled](https://infinitejs.com/posts/avoiding-edge-cases-nodejs-regex-challenges/) — JavaScript regex pitfalls
- [Prompt Drift: The Hidden Failure Mode Undermining Agentic Systems](https://www.comet.com/site/blog/prompt-drift/) — agent prompt drift patterns
- [Writing CLI Tools That AI Agents Actually Want to Use](https://dev.to/uenyioha/writing-cli-tools-that-ai-agents-actually-want-to-use-39no) — interactive prompt blocking in agentic contexts
- [How to Build Multi Agent AI Systems With Context Engineering](https://vellum.ai/blog/multi-agent-systems-building-with-context-engineering) — context window saturation in multi-agent workflows
- [Platform Engineering Maintenance Pitfalls](https://www.cncf.io/blog/2026/01/21/platform-engineering-maintenance-pitfalls-and-smart-strategies-to-stay-ahead/) — upstream divergence and fork maintenance
- [How to fork: Best practices and guide (2024)](https://joaquimrocha.com/2024/09/22/how-to-fork/) — git fork synchronization strategies
- [5 Case Studies on Developer Tool Adoption](https://business.daily.dev/resources/5-case-studies-on-developer-tool-adoption/) — annotation and standards adoption patterns
- [Securing CLI Based AI Agent](https://medium.com/@visrow/securing-cli-based-ai-agent-c36429e88783) — prompt injection in CLI agent orchestration
