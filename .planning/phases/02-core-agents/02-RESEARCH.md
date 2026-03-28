# Phase 2: Core Agents - Research

**Researched:** 2026-03-28
**Domain:** Claude Code agent authoring, Markdown command files, config-gated agent wrappers
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** gsd-prototyper agent defined in `agents/gsd-prototyper.md` — reads PROJECT.md, REQUIREMENTS.md, and ROADMAP.md as context, scopes work to phases specified via `--phases` flag
- **D-02:** Prototyper produces working scaffold code (runnable, demonstrates structure and intent) with @gsd-tags already embedded — not production-ready implementations
- **D-03:** Prototyper follows ARC standard from `get-shit-done/references/arc-standard.md` for all tag placement and syntax
- **D-04:** Prototyper auto-runs `extract-plan` on completion to generate CODE-INVENTORY.md from the annotated prototype
- **D-05:** `prototype` command defined in `commands/gsd/prototype.md` — spawns gsd-prototyper with project context files
- **D-06:** Command supports `--phases N` flag for scoping to specific phase(s)
- **D-07:** PROTOTYPE-LOG.md template captures: what was built, decisions made during prototyping, and open @gsd-todos — written by prototyper on completion
- **D-08:** gsd-code-planner agent defined in `agents/gsd-code-planner.md` — reads CODE-INVENTORY.md as primary input AND scans source @gsd-tags as supplementary input
- **D-09:** Code-planner generates compact Markdown PLAN.md files: tasks, target files, and success criteria — no XML wrappers, no `<research>` sections, no plan-check blocks
- **D-10:** Code-planner plans are compact enough for a single executor pass — no multi-plan phase decomposition unless the annotation scope demands it
- **D-11:** gsd-executor modification implemented as NEW agent `agents/gsd-arc-executor.md` — a wrapper that extends gsd-executor behavior with ARC comment obligations (adds @gsd-decision tags, removes completed @gsd-todo tags)
- **D-12:** gsd-planner modification implemented as NEW agent `agents/gsd-arc-planner.md` — a wrapper that extends gsd-planner to accept @gsd-tags as planning input alongside or instead of requirements docs
- **D-13:** Wrapper agents are the ONLY approach — upstream agent files (gsd-executor.md, gsd-planner.md) remain unmodified for merge compatibility
- **D-14:** Agent modifications gated by `arc.enabled` in `.planning/config.json` — when false, wrapper agents behave identically to originals
- **D-15:** Workflow commands check `config.arc.enabled` and `config.default_phase_mode` to determine which agent variant to spawn

### Claude's Discretion

- Exact prompt structure and instruction ordering within agent .md files
- How the code-planner discovers and reads relevant source files beyond CODE-INVENTORY.md
- PROTOTYPE-LOG.md exact Markdown template layout and section ordering
- How wrapper agents delegate to or extend the behavior of upstream agents (instruction inclusion vs reference)

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope

</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PROT-01 | gsd-prototyper agent builds working prototypes with ARC annotations embedded in code | Agent authoring pattern from gsd-annotator.md is the direct template; ARC standard governs tag embedding |
| PROT-02 | prototype command spawns prototyper with PROJECT.md, REQUIREMENTS.md, ROADMAP.md context | annotate.md command is the direct template; Task tool spawning pattern is established |
| PROT-03 | prototype command supports --phases flag for scoping and auto-runs extract-plan on completion | annotate.md auto-chain pattern + parseNamedArgs() flag handling in gsd-tools.cjs |
| PROT-04 | PROTOTYPE-LOG.md template captures what was built, decisions made, and open @gsd-todos | New Markdown template file; prototyper writes it via Write tool on completion |
| PLAN-01 | gsd-code-planner agent reads CODE-INVENTORY.md and source code @gsd-tags as primary input | arc-scanner.cjs produces CODE-INVENTORY.md; agent reads it via Read tool; supplemental scan via extract-tags CLI |
| PLAN-02 | gsd-code-planner generates compact Markdown plans (no XML, no research, no plan-check) | Distinct from gsd-planner format; compact task+file+success-criteria structure only |
| AMOD-01 | gsd-executor extended with ARC comment obligation (adds @gsd-decision, removes @gsd-todo) | gsd-arc-executor.md wrapper pattern; inherits gsd-executor behavior then adds ARC obligations |
| AMOD-02 | gsd-planner extended with code-based planning mode (reads @gsd-tags as input) | gsd-arc-planner.md wrapper; extends gsd-planner to accept CODE-INVENTORY.md as primary input |
| AMOD-03 | Agent modifications are config-gated to preserve upstream compatibility | config.arc.enabled check via `config-get arc.enabled` at agent start; fallback to upstream behavior when false |

</phase_requirements>

---

## Summary

Phase 2 delivers five new Markdown files: three new agent files (`gsd-prototyper.md`, `gsd-code-planner.md`, one command file (`commands/gsd/prototype.md`), and two wrapper agent files (`gsd-arc-executor.md`, `gsd-arc-planner.md`). No CJS code changes are required — all phase 2 work is agent authoring and command authoring in Markdown. The one config-gating mechanism (checking `arc.enabled`) uses the existing `config-get` CLI that Phase 1 already wired.

The highest-risk deliverable is `gsd-code-planner.md`. STATE.md flags it as HIGH risk because the prompt structure for reliably producing compact PLAN.md from CODE-INVENTORY.md is not proven. The research finding: this risk is best mitigated by giving the code-planner a concrete, locked output format (a Markdown template) and banning the agent from any research, XML, or plan-check sections. The planner must be instructed to treat each `@gsd-todo` as a task, each `@gsd-context`/`@gsd-decision` as background, and produce exactly one PLAN.md of tasks with file targets and success criteria.

The wrapper agent approach (D-11, D-12, D-13) is the correct implementation. Wrappers reference upstream agent behavior via `@agent-file` context includes or inline instruction duplication — the discretion choice is which delegation mechanism to use (see Architecture Patterns section for recommendation).

**Primary recommendation:** Author all five Markdown files following the exact patterns established in Phase 1, with particular care on agent-frontmatter test requirements (anti-heredoc instruction + commented hooks). Register new agents in `model-profiles.cjs` only if they need model profile gating — Phase 1's gsd-annotator was not registered and that is fine for agent-only (non-spawned-by-init) agents.

---

## Standard Stack

### Core

| Component | Format | Purpose | Why |
|-----------|--------|---------|-----|
| Agent files | Markdown `.md` with YAML frontmatter | Define new agents | Established format for all 19 existing agents |
| Command files | Markdown `.md` with YAML frontmatter | Define slash commands | Established format for all ~55 existing commands |
| `arc-scanner.cjs` | Existing CJS module | Tag scanning for code-planner input | Already built in Phase 1; zero new CJS code needed |
| `config-get` CLI | `gsd-tools.cjs config-get <key>` | Read `arc.enabled` for config gating | Existing command; no new CJS needed |

### No New Dependencies

This phase requires zero new npm packages. All functionality is:
- Markdown prose (agent/command authoring)
- Existing `gsd-tools.cjs` CLI calls (config-get, extract-tags)
- Read/Write/Edit/Bash/Task tools used by existing agents

**Installation:** None required.

---

## Architecture Patterns

### Recommended Project Structure for Phase 2

```
agents/
├── gsd-prototyper.md       # NEW — prototyper agent
├── gsd-code-planner.md     # NEW — code-planner agent
├── gsd-arc-executor.md     # NEW — ARC executor wrapper
├── gsd-arc-planner.md      # NEW — ARC planner wrapper
commands/gsd/
└── prototype.md            # NEW — prototype slash command
```

No other files need to be created or modified in Phase 2. `bin/install.js` is Phase 3.

---

### Pattern 1: Agent File Structure (established by gsd-annotator.md)

All agent `.md` files must follow this exact structure to pass `tests/agent-frontmatter.test.cjs`:

```markdown
---
name: gsd-<agent-name>
description: One-line description. Spawned by /gsd:<command> command.
tools: Read, Write, Edit, Bash, Grep, Glob
permissionMode: acceptEdits
color: <color>
# hooks:
#   PostToolUse:
#     - matcher: "Write|Edit"
#       hooks:
#         - type: command
#           command: "..."
---

<role>
You are the GSD <role description>.
...
**ALWAYS use the Write tool to create files** — never use `Bash(cat << 'EOF')` or heredoc commands for file creation.
</role>

<project_context>
...
</project_context>

<execution_flow>
...
</execution_flow>

<constraints>
...
</constraints>
```

**Critical frontmatter rules (verified by tests/agent-frontmatter.test.cjs):**
1. File-writing agents (those with `Write` in tools) MUST include the anti-heredoc instruction: `never use \`Bash(cat << 'EOF')\` or heredoc`
2. File-writing agents MUST have `# hooks:` commented section in frontmatter
3. Agent frontmatter MUST NOT contain a `skills:` field (breaks Gemini CLI)
4. The agent name in the `name:` field must match the filename stem

**Known Phase 1 debt:** `agents/gsd-annotator.md` currently FAILS two of these tests (missing anti-heredoc + missing commented hooks). Phase 2 agents must not repeat these mistakes. The failing tests are pre-existing — do not fix them in this phase.

---

### Pattern 2: Command File Structure (established by annotate.md)

```markdown
---
name: gsd:<command-name>
description: Short description of what the command does
argument-hint: "[path] [--flag VALUE]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Task
  - Glob
  - Grep
---

<objective>
Description with argument documentation.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/<workflow>.md
</execution_context>

<context>
$ARGUMENTS

@.planning/PROJECT.md
@.planning/REQUIREMENTS.md
</context>

<process>
1. Spawn <agent-name> agent via Task tool, passing $ARGUMENTS as context.
2. Wait for agent to complete.
3. Auto-run extract-plan:
   ```bash
   node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" extract-tags --format md --output .planning/prototype/CODE-INVENTORY.md
   ```
4. Show user results summary.
</process>
```

**Key patterns from annotate.md:**
- Includes `Task` in `allowed-tools` — required to spawn sub-agents
- Uses `$HOME` not `~` in bash commands (shell portability)
- Auto-chain to `extract-tags` via Bash after agent completes
- Context block references `$ARGUMENTS` and key project files via `@`-reference syntax

---

### Pattern 3: Config Gating (AMOD-03)

The wrapper agents (gsd-arc-executor.md, gsd-arc-planner.md) must check `arc.enabled` on startup:

```bash
ARC_ENABLED=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" config-get arc.enabled 2>/dev/null || echo "false")
```

When `ARC_ENABLED` is `"true"`, the wrapper adds ARC-specific behavior. When `"false"`, the wrapper behaves identically to the upstream agent. This is the same config-get pattern used by gsd-executor.md for `workflow.auto_advance`.

**Config key already valid:** `arc.enabled` is registered in `VALID_CONFIG_KEYS` in `config.cjs`. No CJS changes needed.

**Default value:** `arc.enabled` defaults to `true` in `buildNewProjectConfig()` in `config.cjs`. However, the current project's `.planning/config.json` does NOT have the `arc` key yet (it was not added during Phase 1's actual config.json update). Phase 2 agents must handle the `key not found` error case from config-get gracefully — treat a missing key as `false` (safe default).

---

### Pattern 4: Wrapper Agent Delegation (Claude's Discretion — Recommendation)

Two options for extending upstream agent behavior without modifying the upstream file:

**Option A: Instruction inclusion (recommended)**
Write the wrapper as a standalone agent whose `<execution_flow>` references the upstream agent's behavior via prose ("Follow all execution steps from gsd-executor...") and adds ARC-specific steps. The wrapper agent is fully self-contained — no runtime file reading of the upstream agent required.

**Option B: Context include at spawn time**
The command spawns the wrapper, and the command file passes the upstream agent's content as context using `@agents/gsd-executor.md`. This is fragile — the path is installation-path-dependent.

**Recommendation: Option A.** Wrappers are self-contained prose files. The ARC-specific behavior is additive: it instructs the executor to check for completed `@gsd-todo` tags in files it touches and remove them, and to add `@gsd-decision` tags for significant design choices made during execution. The gsd-arc-planner wrapper extends gsd-planner by instructing it to read `CODE-INVENTORY.md` as its requirements input when `default_phase_mode` is `code-first`.

---

### Pattern 5: Code-Planner Output Format (PLAN-02)

The gsd-code-planner must produce a distinctly compact plan format — NOT the gsd-planner format (which uses XML task blocks, research sections, plan-check blocks). The compact format:

```markdown
# Code Plan: [Description from @gsd-context tags]

## Context
[Summary of what CODE-INVENTORY.md says — architectural context from @gsd-context tags]

## Tasks

### Task 1: [Name derived from @gsd-todo description]
**Files:** [target files from @gsd-ref or inferred from @gsd-todo location]
**Action:** [What to implement]
**Done when:** [Success criteria]

### Task 2: ...

## Success Criteria
- [ ] [Overall success criterion]
```

The code-planner should treat `@gsd-todo` tags as the task backlog, `@gsd-context` and `@gsd-decision` as background context, `@gsd-constraint` as hard limits on the implementation, and `@gsd-risk` as items requiring special handling.

---

### Pattern 6: Prototyper Scope Control (`--phases N`)

The `prototype` command passes `--phases N` to the prototyper agent via `$ARGUMENTS`. The prototyper reads this flag and filters REQUIREMENTS.md to only requirements whose phase matches N. The prototyper uses `parseNamedArgs()` style reading — no CJS required, the agent parses `$ARGUMENTS` as text context.

---

### Anti-Patterns to Avoid

- **Modifying upstream agents:** gsd-executor.md and gsd-planner.md must not be touched (D-13). All changes go in wrapper files.
- **XML in code-planner output:** The code-planner must produce plain Markdown, not the XML task block format gsd-planner uses (D-09).
- **Hardcoding `arc.enabled: true`:** Wrapper agents must check the config, not assume it (D-14). The current config.json doesn't even have the `arc` section, so the fallback path must work.
- **Forgetting anti-heredoc instruction:** gsd-annotator.md is a live example of what happens when you forget — it fails two tests. All Phase 2 file-writing agents must include the instruction.
- **Using `~` instead of `$HOME` in bash paths:** Existing commands use `$HOME/.claude/...` consistently. Using `~` causes failures in some shell contexts.
- **Not including `Task` in command allowed-tools:** Commands that spawn agents need `Task` in their `allowed-tools` list or agent spawning silently falls back to general-purpose.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Scanning source files for @gsd-tags | Custom file reader in agent | `gsd-tools.cjs extract-tags --format json` | Phase 1 built and tested the scanner |
| Reading config values | Inline JSON parsing in agent | `gsd-tools.cjs config-get arc.enabled` | Existing config-get command handles all edge cases |
| Generating CODE-INVENTORY.md | Custom markdown generator | `gsd-tools.cjs extract-tags --format md --output ...` | Phase 1 built and tested the MD generator |
| Prototype output path | Custom path resolution | `.planning/prototype/` (already established by extract-plan) | Consistent with Phase 1 output conventions |

**Key insight:** All the hard infrastructure work was done in Phase 1. Phase 2 is purely agent prompt authoring — the CJS layer is done.

---

## Common Pitfalls

### Pitfall 1: Missing Anti-Heredoc Instruction (HIGH PROBABILITY)
**What goes wrong:** `agent-frontmatter.test.cjs` fails with "missing anti-heredoc instruction" for new file-writing agents.
**Why it happens:** Phase 1's gsd-annotator.md already demonstrates this — the instruction was omitted and tests now fail for it.
**How to avoid:** Every agent that has `Write` in its tools list must include the phrase `never use \`Bash(cat << 'EOF')\` or heredoc commands for file creation` in its prompt body.
**Warning signs:** Check `node --test tests/agent-frontmatter.test.cjs` after creating each agent file.

### Pitfall 2: Missing Commented Hooks Pattern (HIGH PROBABILITY)
**What goes wrong:** `agent-frontmatter.test.cjs` fails with "missing commented hooks: pattern in frontmatter."
**Why it happens:** Same gap as anti-heredoc — gsd-annotator.md shows the current failure mode.
**How to avoid:** File-writing agents MUST have `# hooks:` commented block in YAML frontmatter (see Pattern 1 above).
**Warning signs:** Test suite reports `gsd-<name> has commented hooks pattern` failure.

### Pitfall 3: Spawn Type Validation Failure
**What goes wrong:** `agent-frontmatter.test.cjs` "named agent spawns use correct agent names" test fails.
**Why it happens:** The test validates that any `subagent_type="gsd-X"` referenced in command/workflow files exists as `agents/gsd-X.md`. If prototype.md references `gsd-prototyper` before `agents/gsd-prototyper.md` exists, or references a wrong name, the test fails.
**How to avoid:** Create the agent `.md` file before or in the same task as the command file that references it.
**Warning signs:** `"${file} references unknown agent type: ${agentType}"` test error.

### Pitfall 4: Config Key Not Found Error
**What goes wrong:** `gsd-tools.cjs config-get arc.enabled` exits with error if `arc` section doesn't exist in `.planning/config.json`.
**Why it happens:** The current project config.json does not have the `arc` key. Phase 1's buildNewProjectConfig() includes it for NEW projects, but this project's config predates that addition.
**How to avoid:** Wrapper agents should use `|| echo "false"` fallback: `ARC_ENABLED=$(... config-get arc.enabled 2>/dev/null || echo "false")`.
**Warning signs:** Agent startup fails immediately with "Key not found: arc.enabled".

### Pitfall 5: Code-Planner Drift to gsd-planner Format
**What goes wrong:** Code-planner produces XML task blocks, research sections, or plan-check blocks like gsd-planner does — violating D-09.
**Why it happens:** If code-planner reads gsd-planner agent as context or is given similar instructions, it adopts the same format.
**How to avoid:** Code-planner prompt must explicitly ban XML wrappers and research sections. Provide the compact PLAN.md template format directly in the agent prompt.
**Warning signs:** Code-planner output contains `<task>`, `<research>`, or `<plan_check>` XML.

### Pitfall 6: Wrapper Agent Reads Upstream Agent File at Runtime
**What goes wrong:** If gsd-arc-executor tries to read gsd-executor.md at runtime via `@agents/gsd-executor.md`, the path resolves differently in installed vs local contexts.
**Why it happens:** The agents directory is at `$HOME/.claude/agents/` when installed, not the project root.
**How to avoid:** Use Option A delegation (self-contained prose, no runtime reads of upstream agent files). The wrapper carries its own complete instructions.

---

## Code Examples

### Config Gating Pattern (from gsd-executor.md)

```bash
# Source: agents/gsd-executor.md auto_mode_detection section
AUTO_CHAIN=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" config-get workflow._auto_chain_active 2>/dev/null || echo "false")
AUTO_CFG=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" config-get workflow.auto_advance 2>/dev/null || echo "false")
```

Apply same pattern for arc.enabled:
```bash
ARC_ENABLED=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" config-get arc.enabled 2>/dev/null || echo "false")
```

### Auto-Chain to extract-plan (from commands/gsd/annotate.md)

```bash
# Source: commands/gsd/annotate.md process step 3
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" extract-tags --format md --output .planning/prototype/CODE-INVENTORY.md
```

### Agent Frontmatter Template (from agents/gsd-executor.md)

```yaml
---
name: gsd-prototyper
description: Builds working code prototypes with ARC annotations embedded. Spawned by /gsd:prototype command.
tools: Read, Write, Edit, Bash, Grep, Glob
permissionMode: acceptEdits
color: cyan
# hooks:
#   PostToolUse:
#     - matcher: "Write|Edit"
#       hooks:
#         - type: command
#           command: "npx eslint --fix $FILE 2>/dev/null || true"
---
```

### Task Tool Spawn Pattern (from commands/gsd/annotate.md)

```markdown
1. **Spawn gsd-prototyper agent** via the Task tool, passing `$ARGUMENTS` as context.
```

Note: The Task tool with a named agent is how commands spawn specialized agents. The agent name must match an existing `agents/gsd-<name>.md` file exactly.

---

## State of the Art

| Old Approach | Current Approach | Impact |
|--------------|------------------|--------|
| Modify upstream gsd-executor.md | Wrapper agent gsd-arc-executor.md (D-11, D-13) | Upstream mergeability preserved |
| Plan-first only workflow | Config-gated code-first mode (arc.enabled) | Existing users unaffected |
| annotate.md as prototype pattern | annotate.md IS the template for prototype.md | Direct reuse; no new patterns |

---

## Open Questions

1. **Should new agents be registered in model-profiles.cjs?**
   - What we know: `EXPECTED_AGENTS` in agent-install-validation.test.cjs = `Object.keys(MODEL_PROFILES)`. gsd-annotator is NOT in MODEL_PROFILES and its absence does not cause test failures (those tests check init command behavior, not every agent file).
   - What's unclear: Whether the model-profiles registration is needed for Phase 2 agents to be "visible" to any installation validation.
   - Recommendation: Add Phase 2 agents to MODEL_PROFILES as a quality step. The correct profile tier: prototyper=sonnet/balanced (heavy work), code-planner=sonnet/balanced (planning-class), arc-executor=sonnet (same as executor), arc-planner=sonnet (same as planner). This is low-risk additive change to one CJS file.

2. **Does config.json need the `arc` section added as a Phase 2 task?**
   - What we know: Current `.planning/config.json` lacks the `arc` key. `buildNewProjectConfig()` includes it for new projects. Wrapper agents check `arc.enabled` at startup.
   - What's unclear: Whether there is a migration path (a `config-ensure-section` style command) or if the wrapper agent's `|| echo "false"` fallback is sufficient.
   - Recommendation: The `|| echo "false"` fallback is sufficient for Phase 2. The config migration (adding `arc` to existing projects) is a Phase 3 concern (or a simple manual `config-set arc.enabled true`).

3. **Where does PROTOTYPE-LOG.md live?**
   - What we know: D-07 says the prototyper writes it on completion. D-04 says prototyper auto-runs extract-plan which writes to `.planning/prototype/CODE-INVENTORY.md`.
   - What's unclear: Exact path for PROTOTYPE-LOG.md — `.planning/prototype/PROTOTYPE-LOG.md` would be consistent with the prototype output directory.
   - Recommendation (Claude's Discretion): Write to `.planning/prototype/PROTOTYPE-LOG.md`. This collocates it with CODE-INVENTORY.md.

---

## Environment Availability

Step 2.6: SKIPPED — Phase 2 is purely Markdown file authoring with no external dependencies beyond Node.js and the existing gsd-tools.cjs CLI (already confirmed operational in Phase 1).

---

## Validation Architecture

> `workflow.nyquist_validation` is `false` in `.planning/config.json` — this section is included for awareness only; automated test gating is not enforced for this phase.

### Test Framework
| Property | Value |
|----------|-------|
| Framework | `node:test` (built-in) + `node:assert` |
| Config file | No config file — run directly via `node --test` |
| Quick run command | `node --test tests/agent-frontmatter.test.cjs` |
| Full suite command | `node --test tests/*.test.cjs` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PROT-01 | gsd-prototyper.md has valid frontmatter (anti-heredoc, commented hooks, no skills:) | unit | `node --test tests/agent-frontmatter.test.cjs` | Yes |
| PROT-02 | prototype.md references valid agent type gsd-prototyper | unit | `node --test tests/agent-frontmatter.test.cjs` | Yes |
| PROT-03 | prototype command auto-chains extract-tags (manual verification only) | manual | N/A | N/A |
| PLAN-01 | gsd-code-planner.md has valid frontmatter | unit | `node --test tests/agent-frontmatter.test.cjs` | Yes |
| PLAN-02 | Code-planner produces compact format (prompt content only, not automated) | manual | N/A | N/A |
| AMOD-01 | gsd-arc-executor.md has valid frontmatter | unit | `node --test tests/agent-frontmatter.test.cjs` | Yes |
| AMOD-02 | gsd-arc-planner.md has valid frontmatter | unit | `node --test tests/agent-frontmatter.test.cjs` | Yes |
| AMOD-03 | Config gating via arc.enabled (prose instruction, not automated) | manual | N/A | N/A |

### Sampling Rate
- **Per task commit:** `node --test tests/agent-frontmatter.test.cjs`
- **Per wave merge:** `node --test tests/agent-frontmatter.test.cjs tests/commands.test.cjs`
- **Phase gate:** All agent-frontmatter tests pass (currently 88 pass, 2 fail — new agents must not add failures)

### Wave 0 Gaps
None — existing test infrastructure covers all automated Phase 2 requirements. No new test files needed.

---

## Project Constraints (from CLAUDE.md)

All directives below are hard constraints the planner must enforce:

| Constraint | Implication for Phase 2 |
|------------|------------------------|
| Zero runtime dependencies | No npm packages added — all new code is Markdown prose |
| node:test for CJS tests | If any CJS is added (none expected), it uses node:test |
| Tech stack: JavaScript/Node.js, Markdown, JSON | Phase 2 = Markdown only; no new file types |
| All original GSD commands must continue working unchanged | Wrapper agents must not modify any existing agent/command files |
| Upstream mergeability | D-13 enforces this — no upstream file modifications |
| GSD Workflow Enforcement | Phase 2 work goes through /gsd:execute-phase |

---

## Sources

### Primary (HIGH confidence)
- `agents/gsd-annotator.md` — Direct template for gsd-prototyper.md (verified by reading)
- `commands/gsd/annotate.md` — Direct template for prototype.md (verified by reading)
- `agents/gsd-executor.md` — Base behavior for gsd-arc-executor.md wrapper (verified by reading)
- `agents/gsd-planner.md` — Base behavior for gsd-arc-planner.md wrapper (verified by reading)
- `get-shit-done/references/arc-standard.md` — ARC tag syntax, all 8 types (verified by reading)
- `get-shit-done/bin/lib/config.cjs` — arc.enabled key, config-get pattern (verified by reading)
- `get-shit-done/bin/lib/arc-scanner.cjs` — TAG_LINE_RE, VALID_TAG_TYPES (verified by reading)
- `get-shit-done/bin/lib/model-profiles.cjs` — EXPECTED_AGENTS = Object.keys(MODEL_PROFILES) (verified)
- `tests/agent-frontmatter.test.cjs` — Exact test assertions for agent file requirements (verified + executed)
- `node --test tests/agent-frontmatter.test.cjs` — Live test run: 88 pass, 2 fail (gsd-annotator defects)

### Secondary (MEDIUM confidence)
- `.planning/STATE.md` — gsd-code-planner flagged HIGH risk; wrapper pattern decision recorded
- `.planning/phases/02-core-agents/02-CONTEXT.md` — All locked decisions verified

---

## Metadata

**Confidence breakdown:**
- Agent authoring patterns: HIGH — read all 5 canonical files, ran tests
- Config gating pattern: HIGH — read config.cjs, confirmed VALID_CONFIG_KEYS, verified default value
- Code-planner output format: MEDIUM — format is Claude's design decision, not derived from existing code; compact template recommendation is inference from locked decisions D-09/D-10
- Test coverage: HIGH — ran agent-frontmatter tests live; confirmed what passes and what fails

**Research date:** 2026-03-28
**Valid until:** 2026-04-28 (stable — agent patterns change rarely)
