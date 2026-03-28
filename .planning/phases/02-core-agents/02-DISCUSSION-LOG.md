# Phase 2: Core Agents - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md -- this log preserves the alternatives considered.

**Date:** 2026-03-28
**Phase:** 02-core-agents
**Areas discussed:** Prototyper scope, Code-planner format, Agent modification strategy, Prototype log, Config gating
**Mode:** Auto (--auto flag)

---

## Prototyper Scope and Behavior

| Option | Description | Selected |
|--------|-------------|----------|
| Phase-scoped from ROADMAP.md requirements | Reads PROJECT.md + REQUIREMENTS.md + ROADMAP.md, scopes to --phases argument | [auto] |
| Full codebase analysis | Prototyper analyzes entire codebase before building | |
| User-guided interactive | Prototyper asks user what to build at each step | |

**User's choice:** [auto] Phase-scoped from ROADMAP.md requirements (recommended default)
**Notes:** Aligns with PROT-02 requirement. Matches annotator pattern from Phase 1.

---

## Prototyper Output Style

| Option | Description | Selected |
|--------|-------------|----------|
| Working scaffolds with ARC annotations | Runnable code demonstrating structure and intent, not production-ready | [auto] |
| Production-ready implementations | Fully polished code ready for deployment | |
| Skeleton stubs only | File structure and function signatures without logic | |

**User's choice:** [auto] Working scaffolds with ARC annotations (recommended default)
**Notes:** Scaffolds provide enough substance for code-planner to extract meaningful plans while keeping prototype scope manageable.

---

## Code-Planner Input Sources

| Option | Description | Selected |
|--------|-------------|----------|
| Both CODE-INVENTORY.md + source tags | Primary from inventory, supplementary from scanning actual source | [auto] |
| CODE-INVENTORY.md only | Rely solely on extracted inventory | |
| Source tags only | Scan source directly, skip inventory | |

**User's choice:** [auto] Both (recommended default per PLAN-01)
**Notes:** CODE-INVENTORY.md provides the structured overview; source tags provide contextual detail around each annotation.

---

## Code-Planner Plan Format

| Option | Description | Selected |
|--------|-------------|----------|
| Compact Markdown (no XML, no research) | Tasks, files, success criteria only | [auto] |
| Standard GSD plan format | Full PLAN.md with all sections | |
| Minimal task list | Just a checklist of tasks | |

**User's choice:** [auto] Compact Markdown (recommended default per PLAN-02)
**Notes:** Compact format reduces overhead and reflects code-first philosophy. Plans should be actionable, not documentary.

---

## Agent Modification Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| New wrapper agents | gsd-arc-executor.md and gsd-arc-planner.md as separate files | [auto] |
| Config-gated patches | Modify existing agents with conditional behavior | |
| Runtime agent composition | Dynamically compose agent behavior at spawn time | |

**User's choice:** [auto] New wrapper agents (recommended -- aligns with STATE.md decision)
**Notes:** Locked decision from Phase 1 research. Preserves upstream files untouched for merge compatibility.

---

## Config Gating Mechanism

| Option | Description | Selected |
|--------|-------------|----------|
| arc.enabled check at agent startup | Simple boolean gate in config.json | [auto] |
| Per-command flag override | --arc flag on each command | |
| Phase mode auto-detection | Check phase_modes config for current phase | |

**User's choice:** [auto] arc.enabled check (recommended default per AMOD-03)
**Notes:** Simplest approach. Per-command flags can be added later if needed.

---

## Claude's Discretion

- Exact prompt structure within agent .md files
- How code-planner discovers relevant source files
- PROTOTYPE-LOG.md template layout
- How wrapper agents reference/extend upstream agent behavior

## Deferred Ideas

None -- discussion stayed within phase scope
