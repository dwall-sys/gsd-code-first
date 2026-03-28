# Phase 3: Workflow, Distribution, and Docs - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md -- this log preserves the alternatives considered.

**Date:** 2026-03-28
**Phase:** 03-workflow-distribution-and-docs
**Areas discussed:** Iterate loop, Approval gate, Mode config, Installer, Documentation
**Mode:** Auto (--auto flag)

---

## Iterate Loop Orchestration

| Option | Description | Selected |
|--------|-------------|----------|
| Sequential pipeline via gsd-tools.cjs subcommand | extract-tags -> code-planner -> approval -> executor | [auto] |
| Parallel agent chain | Run multiple steps concurrently | |
| Single monolithic agent | One agent handles entire loop | |

**User's choice:** [auto] Sequential pipeline (recommended default)
**Notes:** Each step's output feeds the next. Matches existing auto-chain patterns from annotate and prototype commands.

---

## Approval Gate Design

| Option | Description | Selected |
|--------|-------------|----------|
| Interactive prompt with --non-interactive flag | Pause, show plan, wait for approval. CI flag auto-approves. | [auto] |
| Always auto-approve | No human gate | |
| Config-based approval setting | Toggle in config.json | |

**User's choice:** [auto] Interactive prompt with --non-interactive (recommended default per ITER-03)
**Notes:** Approval gate is table stakes per Out of Scope decision ("Automatic commit/push on iteration destroys developer trust").

---

## Iterate Flags

| Option | Description | Selected |
|--------|-------------|----------|
| --verify and --annotate per ITER-02 | Verify after execution, refresh tags after execution | [auto] |
| Single --post-process flag | Bundle all post-processing | |
| No additional flags | Keep iterate minimal | |

**User's choice:** [auto] --verify and --annotate (recommended default per ITER-02)
**Notes:** Separate flags give fine-grained control. --annotate refreshes tags for the next iterate cycle.

---

## set-mode Command Location

| Option | Description | Selected |
|--------|-------------|----------|
| New gsd-tools.cjs subcommand + slash command | Subcommand for programmatic use, slash command for user | [auto] |
| Slash command only | No CLI subcommand | |
| Config-set only | Use existing config-set directly | |

**User's choice:** [auto] Subcommand + slash command (recommended default)
**Notes:** Subcommand provides validation (only code-first/plan-first/hybrid accepted). Slash command provides discoverability.

---

## deep-plan Composition

| Option | Description | Selected |
|--------|-------------|----------|
| Slash command chaining discuss-phase then plan-phase | Simple composition of two existing commands | [auto] |
| New agent | Dedicated planning agent | |
| gsd-tools.cjs pipeline | Programmatic chain | |

**User's choice:** [auto] Slash command composition (recommended default per MODE-03)
**Notes:** Minimal new code. Leverages existing commands.

---

## Installer Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Add to existing copy loops with GSD_CF_ markers | Extend current install.js pattern | [auto] |
| Separate installer script | New install-cf.js | |
| npm postinstall hook | Auto-copy on npm install | |

**User's choice:** [auto] Extend existing install.js (recommended default per DIST-01/03)
**Notes:** GSD_CF_ namespace prevents conflicts with upstream. Additive change to existing installer.

---

## Documentation Structure

| Option | Description | Selected |
|--------|-------------|----------|
| README.md + user guide section | Single document with installation, workflow, and guide | [auto] |
| Separate docs directory | Multiple files in docs/ | |
| README.md only | Minimal docs | |

**User's choice:** [auto] README.md + user guide section (recommended default per DOCS-02/03)
**Notes:** Single entry point for new developers. References arc-standard.md for tag details.

---

## Claude's Discretion

- Iterate command internal orchestration
- set-mode output formatting
- deep-plan flag passthrough
- README structure and examples
- Help command discovery mechanism

## Deferred Ideas

None -- discussion stayed within phase scope
