---
phase: prototype
plan: code-plan
subsystem: brainstorm-command
tags: [brainstorm, prd-generation, conversational-agent, command-orchestrator]
dependency_graph:
  requires: []
  provides: ["/gsd:brainstorm command", "gsd-brainstormer agent"]
  affects: ["/gsd:prototype (PRD consumption)", "BRAINSTORM-LEDGER.md"]
tech_stack:
  added: []
  patterns: [delimiter-based-output-parsing, read-then-append-ledger, collision-resistant-slugs, terse-user-probe]
key_files:
  created:
    - commands/gsd/brainstorm.md
    - agents/gsd-brainstormer.md
  modified: []
decisions:
  - "Delimiter-based output parsing over JSON for robustness with LLM-generated content"
  - "4-char hex hash suffix on slugs for collision resistance without excessive filename length"
  - "Read-then-append ledger pattern using single atomic Write call (not true append)"
  - "Terse-user probe threshold set at 10 words to prevent thin PRDs"
  - "3-branch independence heuristic: coupled/disjoint/mixed with user choice on mixed"
metrics:
  duration: "3m 28s"
  completed: "2026-03-30T11:03:35Z"
  tasks_completed: 2
  tasks_total: 2
---

# Code Plan: Brainstorm Command and Agent -- Summary

Complete implementation of `/gsd:brainstorm` command orchestrator and `gsd-brainstormer` conversational agent with delimiter-based output parsing, collision-resistant slug generation, terse-user probing, and 3-branch independence heuristic for multi-PRD decisions.

## Tasks Completed

### Task 1: Complete `commands/gsd/brainstorm.md`

**Commit:** c5598a2

Filled in all prose gaps in the command orchestrator:

- **Step 2 (output parsing):** Added full delimiter extraction logic for 8 named blocks (MULTI_PRD, PRD CONTENT, PRD FILE, FEATURE GROUPS, DECISIONS, EXCLUSIONS, DEFERRED, COUNTS). Each block is parsed between its `===` delimiters and stored in named variables for Steps 3-6.
- **Step 3 (correction loop):** Specified that follow-up Task() calls include the ORIGINAL context plus user corrections appended under a `**Corrections from user:**` header. Restart path clears all state and spawns fresh.
- **Step 4 (slug generation):** Added 6-step slug generation rule: lowercase, replace specials with hyphens, collapse consecutive hyphens, trim, truncate to 40 chars at word boundary, append 4-char MD5 hex hash of full title.
- **Step 5 (ledger append):** Added explicit read-then-append pattern: check existence, read existing content if present, write complete file (old + new session block) in single Write call. ISO 8601 timestamp via `date -u`.

All 6 `@gsd-todo(ref:AC-N)` tags verified present.

### Task 2: Complete `agents/gsd-brainstormer.md`

**Commit:** 2277498

Filled in all prose gaps in the conversational agent:

- **Step 2 (terse-user probe):** Added explicit rule: if answer is fewer than 10 words, follow up with "Can you give me a concrete example?" before moving on. Added exact 6-10 question transition phrase.
- **Step 3 (independence heuristic):** Added 3-branch decision tree: (1) coupled = single PRD, (2) disjoint = separate PRDs, (3) mixed = present both options and ask user. Added `--multi` flag early-suggest behavior.
- **Step 4 (AC self-check):** Added vague-language filter scanning for "should", "might", "consider", "gracefully", "properly", "appropriately", "reasonable", "adequate" without concrete qualifiers. Added `## Dependencies` section requirement for multi-PRD mode.
- **Step 5 (counts block):** Added `=== COUNTS ===` block after `=== END BRAINSTORM OUTPUT ===` with 7 key=value count lines for command layer consumption.

All @gsd-* annotations preserved. All 8 constraints unchanged.

## Deviations from Plan

None -- plan executed exactly as written.

## Success Criteria Verification

- [x] All 6 `@gsd-todo(ref:AC-N)` tags in `brainstorm.md` are addressed by concrete implementation prose (AC-1 through AC-6)
- [x] `commands/gsd/brainstorm.md` has no undefined variables, TBD placeholders, or ambiguous control flow after Task() returns
- [x] `agents/gsd-brainstormer.md` has no gaps in Steps 2-5
- [x] PRD output format enforced in agent Step 4 and matches `/gsd:prototype` extraction
- [x] Ledger append pattern explicit in command Step 5
- [x] Slug collision-resistance rule explicit in command Step 4
- [x] BRAIN-01 through BRAIN-07 all satisfied

## Known Stubs

None -- these are agent/command markdown files (prose instructions), not executable code. No stubs, hardcoded empty values, or placeholder data.

## Self-Check: PASSED
