---
name: gsd:deep-plan
description: Chain discuss-phase then plan-phase for phases needing upfront reasoning (Code-First fork)
argument-hint: "<phase-number> [flags]"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Task
  - Glob
  - Grep
---

<objective>
Chains `/gsd:discuss-phase` followed by `/gsd:plan-phase` for phases where upfront reasoning is valuable before code-first iteration. Use deep-plan when a phase has significant unknowns or architectural decisions that benefit from structured discussion before building.

**Arguments:**
- `phase-number` — the phase number to plan (e.g., `3`)
- All other flags are passed through to both discuss-phase and plan-phase

Deep-plan is a gsd-code-first convenience command. The two constituent commands remain independently usable.
</objective>

<context>
$ARGUMENTS

@.planning/PROJECT.md
@.planning/ROADMAP.md
</context>

<process>

1. **Extract phase number** from `$ARGUMENTS` (first positional argument).

2. **Run `/gsd:discuss-phase`** with `$ARGUMENTS` (all flags pass through):
   ```
   /gsd:discuss-phase $ARGUMENTS
   ```
   This produces a `CONTEXT.md` for the phase capturing open questions, research findings, and architectural decisions.

3. **Wait for discuss-phase to complete** and confirm that `CONTEXT.md` was produced in the phase directory.

4. **Run `/gsd:plan-phase`** with `$ARGUMENTS` (all flags pass through):
   ```
   /gsd:plan-phase $ARGUMENTS
   ```
   This reads the `CONTEXT.md` produced in step 2 and generates `PLAN.md` files for the phase.

5. **Show summary:** "Deep plan complete for phase N. CONTEXT.md and PLAN.md(s) created."

</process>
