# Phase 4: Tech Debt Cleanup - Research

**Researched:** 2026-03-28
**Domain:** GSD Code-First fork internal files — command frontmatter, agent files, README documentation
**Confidence:** HIGH

## Summary

Phase 4 is a targeted cleanup of five known tech debt items identified during the v1.0 milestone audit. All items are non-blocking for the v1 release but degrade developer experience: stale workflow file references produce confusing error context when commands fail, failing tests undermine CI confidence, and undocumented routing limitations lead to user confusion when `arc.enabled` does not affect `/gsd:execute-phase` or `/gsd:plan-phase` as expected.

The three work streams are fully independent and each is a small, surgical edit. No new functionality is introduced — the phase closes existing gaps only. The domain is entirely internal markdown files in `commands/gsd/`, `agents/`, and `README.md`. Zero changes to `gsd-tools.cjs` or any `.cjs` runtime code are required.

**Primary recommendation:** Three targeted edits — (1) remove four stale `<execution_context>` lines from command files, (2) add two missing frontmatter/role patterns to `gsd-annotator.md`, (3) append a Known Limitations section to `README.md`. Each edit is fully self-contained and verifiable with the existing test suite.

---

## Standard Stack

This phase uses no external libraries. All changes are to Markdown files.

### Core
| Technology | Version | Purpose |
|------------|---------|---------|
| Node.js built-in `node:test` | >=20 | Test runner for `agent-frontmatter.test.cjs` |
| Markdown | — | All changed files are `.md` |

### Installation
No installation required. All dependencies already present.

---

## Architecture Patterns

### Recommended Project Structure (unchanged)
```
agents/
  gsd-annotator.md          # needs 2 pattern additions
commands/gsd/
  annotate.md               # needs <execution_context> removed
  prototype.md              # needs <execution_context> removed
  extract-plan.md           # needs <execution_context> removed
README.md                   # needs Known Limitations section added
tests/
  agent-frontmatter.test.cjs  # verification oracle — do not modify
```

### Pattern 1: Agent File-Writing Pattern (anti-heredoc + commented hooks)

All agents that include `Write` in their `tools:` frontmatter field are classified as "file-writing agents" by the test suite. The test detects this by checking `toolsMatch[1].includes('Write')`. File-writing agents MUST include both:

**1. Commented hooks block in frontmatter** — placed immediately after the last frontmatter key, before the closing `---`:
```yaml
# hooks:
#   PostToolUse:
#     - matcher: "Write|Edit"
#       hooks:
#         - type: command
#           command: "npx eslint --fix $FILE 2>/dev/null || true"
```
The test checks: `frontmatter.includes('# hooks:')` where `frontmatter = content.split('---')[1]`.

**2. Anti-heredoc instruction in file body** — placed in the agent's `<role>` section:
```
**ALWAYS use the Write tool to create files** -- never use `Bash(cat << 'EOF')` or heredoc commands for file creation.
```
The test checks: `content.includes("never use \`Bash(cat << 'EOF')\` or heredoc")`.

**Reference implementations:** `agents/gsd-prototyper.md` (line 18) and `agents/gsd-arc-executor.md` (line 26) both pass these checks and serve as exact templates.

### Pattern 2: Command File `<execution_context>` Removal

Three command files (`annotate.md`, `prototype.md`, `extract-plan.md`) contain `<execution_context>` blocks that reference workflow files that do not exist at the installed path `~/.claude/get-shit-done/workflows/`.

Current state (example from `annotate.md`):
```
<execution_context>
@~/.claude/get-shit-done/workflows/annotate.md
</execution_context>
```

The audit finding: "All 3 commands have self-contained `<process>` sections and execute correctly despite missing workflow files." The `<process>` section is the actual execution logic. The `<execution_context>` block is vestigial — it was designed to load a separate workflow file that was never created.

**Fix:** Remove the entire `<execution_context>...</execution_context>` block from each of the three files. Do not modify the `<process>` section. Do not modify the `<context>` section.

Verification: No test currently asserts absence of `<execution_context>` blocks. Verification is manual inspection that the block is absent from each file after editing.

### Pattern 3: README Known Limitations Documentation

The README.md uses a "fork section prepended before upstream content" architecture (Decision D-21 from STATE.md). The prepended fork section ends with `---` followed by the upstream content.

**Fix location:** Add a `## Known Limitations` section inside the prepended fork section (before the `---` horizontal rule separator), after the "Code-First Commands" table. This placement:
- Keeps it visible to code-first users who installed the fork
- Does not modify or interleave with upstream content
- Maintains upstream merge compatibility

**Required content** per phase success criteria:
```markdown
## Known Limitations

### ARC Wrapper Agents: Only Reachable via `/gsd:iterate`

When `arc.enabled` is `true` in your project config, the ARC wrapper agents
(`gsd-arc-executor` and `gsd-arc-planner`) are only invoked by the
`/gsd:iterate` command. The traditional GSD entry points `/gsd:execute-phase`
and `/gsd:plan-phase` always spawn the standard `gsd-executor` and
`gsd-planner` regardless of the `arc.enabled` setting.

**Workaround:** Use `/gsd:iterate` for code-first workflows where ARC
annotation obligations are needed. Use `/gsd:execute-phase` or
`/gsd:plan-phase` for plan-first workflows where ARC obligations are not
required.

This limitation is tracked for resolution in v1.1.
```

### Anti-Patterns to Avoid

- **Do not modify `tests/agent-frontmatter.test.cjs`** — the tests are the oracle. Fix the agents to pass the tests, not the other way around.
- **Do not remove `<process>` sections** from command files — those ARE the working execution logic.
- **Do not add `permissionMode: acceptEdits` to gsd-annotator** — that permission is only required for worktree-spawned agents (`gsd-executor`, `gsd-debugger`) per the PERM test suite. gsd-annotator already has `permissionMode: acceptEdits` in its frontmatter (line 6), which is fine but unrelated to the test failures.
- **Do not modify upstream workflow files** in `get-shit-done/workflows/` — those are upstream files, not part of this fork's changes.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead |
|---------|-------------|-------------|
| Determining exact test assertion text | Guess at the string | Read `tests/agent-frontmatter.test.cjs` lines 39, 86 — exact strings are there |
| Finding what "file-writing agent" means | Re-derive definition | Test file line 24-28 — classification is `toolsMatch[1].includes('Write')` |

---

## Common Pitfalls

### Pitfall 1: Partial `<execution_context>` Removal
**What goes wrong:** Removing only the `@~/.claude/...` line but leaving the `<execution_context>` and `</execution_context>` tags, or leaving blank lines.
**Why it happens:** Editors that remove only the content line, not the wrapping tags.
**How to avoid:** Remove the entire block including the opening tag, the reference line, and the closing tag. Verify with `grep -n 'execution_context' commands/gsd/annotate.md` returns no results.
**Warning signs:** File still contains `<execution_context>` after edit.

### Pitfall 2: Frontmatter vs. Body Placement of Hooks
**What goes wrong:** Placing the `# hooks:` commented block in the agent body (below the second `---`) instead of in the frontmatter (between the first and second `---`).
**Why it happens:** The frontmatter is the YAML block between the first and second `---`. The test splits on `---` and checks `content.split('---')[1]`.
**How to avoid:** The `# hooks:` block must appear before the closing `---` of the frontmatter section. Reference `gsd-prototyper.md` lines 7-12 for exact placement.
**Warning signs:** Test still fails after adding hooks block — it's in the wrong section.

### Pitfall 3: Anti-Heredoc Instruction Exact String Mismatch
**What goes wrong:** Adding anti-heredoc instruction with slightly different wording that doesn't match the test assertion.
**Why it happens:** Paraphrasing instead of copying exactly.
**How to avoid:** The test checks for the exact string: `never use \`Bash(cat << 'EOF')\` or heredoc`. Use backticks around `Bash(cat << 'EOF')`. Copy from a passing agent like `gsd-prototyper.md` line 18, or `gsd-arc-executor.md` line 26.
**Warning signs:** Test still fails after adding instruction — check for exact character match including backtick placement.

### Pitfall 4: README Placement Breaks Upstream Merge Compatibility
**What goes wrong:** Adding Known Limitations content after the `---` separator, interleaving with upstream README content.
**Why it happens:** Not noticing the `---` separator that marks the boundary between fork content and upstream content.
**How to avoid:** Add the section BEFORE the `---` horizontal rule that has the comment "The sections below are from the original GSD framework."
**Warning signs:** The new section appears below the "GET SHIT DONE" header when rendered.

---

## Code Examples

### Exact Anti-Heredoc Instruction (copy verbatim)
```markdown
**ALWAYS use the Write tool to create files** -- never use `Bash(cat << 'EOF')` or heredoc commands for file creation.
```
Source: `agents/gsd-prototyper.md` line 18 (verified passing agent)

### Exact Hooks Block for Frontmatter (copy verbatim)
```yaml
# hooks:
#   PostToolUse:
#     - matcher: "Write|Edit"
#       hooks:
#         - type: command
#           command: "npx eslint --fix $FILE 2>/dev/null || true"
```
Source: `agents/gsd-prototyper.md` lines 7-12 (verified passing agent)

### Test Verification Command
```bash
node --test tests/agent-frontmatter.test.cjs 2>&1 | grep -E "(✖|✔|fail)" | head -20
```
Expected after fix: `gsd-annotator has anti-heredoc instruction` shows ✔, `gsd-annotator has commented hooks pattern` shows ✔, `ℹ fail 0`.

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `<execution_context>` as workflow loader | Self-contained `<process>` sections | Phase 1-2 implementation | Workflow loader pattern was designed but workflow files were never created; `<process>` is the actual execution path |

**Deprecated/outdated:**
- `<execution_context>` blocks in `annotate.md`, `prototype.md`, `extract-plan.md`: Reference workflow files that don't exist; safe to remove because `<process>` sections contain all execution logic.

---

## Open Questions

1. **Should `<execution_context>` pattern be used in future commands?**
   - What we know: Three existing command files use it but the target files don't exist
   - What's unclear: Whether this was ever intended to be implemented as separate workflow files
   - Recommendation: Remove for now. If workflow file loading is desired in the future, it should be a deliberate decision with actual files created.

2. **Should gsd-annotator have `permissionMode: acceptEdits`?**
   - What we know: It already has it (line 6 of the agent file). The failing tests are about anti-heredoc and hooks, not permissionMode.
   - What's unclear: Nothing — this is already correct and requires no change.
   - Recommendation: No action needed.

---

## Environment Availability

Step 2.6: SKIPPED — this phase is purely Markdown file edits. No external tools, services, runtimes, or databases are required beyond Node.js (already confirmed present) for test verification.

---

## Validation Architecture

nyquist_validation is explicitly `false` in `.planning/config.json`. This section is skipped.

**Test verification is still required** via the existing `agent-frontmatter.test.cjs` suite, but no new test files are needed. The existing tests ARE the acceptance criteria for tech debt items 1 and 2.

**Test commands:**
- Quick run: `node --test tests/agent-frontmatter.test.cjs`
- Verify fix: `node --test tests/agent-frontmatter.test.cjs 2>&1 | grep -E "(✖|fail)"`
- Expected after fix: `ℹ fail 0`

---

## Project Constraints (from CLAUDE.md)

- **Tech stack**: JavaScript/Node.js, Markdown, JSON only — this phase is Markdown-only, compliant
- **Compatibility**: All original GSD commands must continue working unchanged — no original commands are modified
- **Distribution**: Must be installable via `npx gsd-code-first@latest` — no distribution changes
- **Upstream mergeability**: Primarily additive changes — README addition is additive; command file removals are in fork-only files
- **Zero runtime dependencies**: No new dependencies — this phase adds none
- **Test pattern**: `node:test` for CJS tests — no new tests needed; existing tests verified
- **Never use heredoc via Bash**: Applies to implementation; all file writes in execution must use Write tool

---

## Sources

### Primary (HIGH confidence)
- `tests/agent-frontmatter.test.cjs` — exact test assertions read directly (lines 39, 86); classification logic read directly (lines 24-28)
- `agents/gsd-arc-executor.md` — verified passing agent with both required patterns
- `agents/gsd-prototyper.md` — verified passing agent with both required patterns
- `commands/gsd/annotate.md` — stale `<execution_context>` block confirmed by direct read
- `commands/gsd/prototype.md` — stale `<execution_context>` block confirmed by direct read
- `commands/gsd/extract-plan.md` — stale `<execution_context>` block confirmed by direct read
- `.planning/v1.0-MILESTONE-AUDIT.md` — source of all tech debt items
- `README.md` — fork section structure confirmed; Known Limitations placement verified

### Secondary (MEDIUM confidence)
- Bash test run `node --test tests/agent-frontmatter.test.cjs` — confirmed exactly 2 failures, both on `gsd-annotator`

---

## Metadata

**Confidence breakdown:**
- Stale reference fix: HIGH — files read directly, exact blocks identified, no ambiguity
- Test fixes: HIGH — test assertions read directly, exact strings required known, reference implementations identified
- README documentation: HIGH — placement constraints clear from file structure, success criteria explicit in phase description
- Overall: HIGH — all three work streams are direct file edits with no external dependencies or ambiguity

**Research date:** 2026-03-28
**Valid until:** 2026-04-28 (stable internal file domain)
