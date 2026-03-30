# Stack Research

**Domain:** GSD Code-First Fork — v1.2 Additions (Brainstorm & Feature Map)
**Researched:** 2026-03-29
**Scope:** NEW capabilities only — brainstorm command, FEATURES.md auto-aggregation, architecture mode for /gsd:prototype. Existing v1.0 and v1.1 stack decisions are validated and unchanged.
**Confidence:** HIGH

---

## Ruling Constraint (Unchanged)

The existing codebase is pure Node.js CommonJS (`.cjs`) with **zero runtime npm dependencies**. All v1.2 additions MUST maintain this constraint. This is non-negotiable and has held through v1.0 and v1.1 without any regression.

---

## v1.1 Stack Baseline (Carry-Forward)

No changes to the inherited stack. The table below is a carry-forward reference only — these decisions are not being revisited.

| Component | Decision | Status | Current Version |
|-----------|----------|--------|-----------------|
| Node.js built-ins (CJS layer) | Zero runtime deps | Unchanged | Node.js >=20 |
| `node:test` + `node:assert` | CJS test runner | Unchanged | Built-in >=20 |
| `c8` | Coverage for CJS | Unchanged | 11.0.0 |
| `vitest` | SDK TypeScript tests only | Unchanged | 4.1.2 |
| `@anthropic-ai/claude-agent-sdk` | Programmatic agent invocation | Unchanged | 0.2.87 |
| `esbuild` | Hooks bundling | Unchanged | 0.24.0 |
| Claude Code agent frontmatter (YAML) | Agent file format | Unchanged | Current Claude Code format |
| `arc-scanner.cjs` | Regex tag extraction | Unchanged | Internal module |

---

## v1.2 New Capabilities — Stack Decisions

### 1. `/gsd:brainstorm` Command

**What it does:** Runs an interactive conversation with the developer to scope a feature, explore constraints, and produces one or more structured PRD files in `.planning/` with acceptance criteria, feature groupings, and dependency analysis.

**Stack decision: New command file (`brainstorm.md`) + new agent file (`gsd-brainstormer.md`). Zero new dependencies.**

This follows the identical pattern established by `prototype.md` + `gsd-prototyper.md`:
- The command orchestrates conversation flow (AskUserQuestion loops, approval gates)
- The agent does the structured output work (PRD generation, AC formatting)

**PRD file format — use existing Markdown + frontmatter YAML header:**

PRDs produced by brainstorm MUST use this structure so downstream tools (prototype, feature map) can parse them reliably:

```markdown
---
title: [Feature name]
version: 1.0
created: [ISO date]
phase: [N or null]
status: draft|approved
---

# [Feature name]

## Goal
[One paragraph]

## Acceptance Criteria
AC-1: [imperative statement]
AC-2: [imperative statement]

## Feature Groups
- Group: [name] — [AC-1, AC-2, ...]

## Dependencies
- Depends on: [AC-X from other-PRD.md or none]

## Out of Scope
- [explicit exclusion]
```

Why this format:
- YAML frontmatter is parseable by the agent via LLM reading + by `gsd-tools.cjs` via simple line-by-line scan (no library needed — `status:` and `phase:` are single-value keys resolvable with `line.split(':')[1].trim()`)
- `## Acceptance Criteria` as a labeled section header lets the existing AC extraction logic in `prototype.md` (Step 2) work without modification — it already extracts "AC-N: ..." patterns from prose
- Feature Groups section enables FEATURES.md aggregation (Section 2 below)
- No new file format introduced — existing `.planning/PRD.md` format extended, not replaced

**Output path:** PRDs land in `.planning/PRD-[slug].md` (or `.planning/PRD.md` for single-feature projects). The brainstorm command uses `generate-slug` (already in `gsd-tools.cjs`) to derive the slug from the feature title.

**Conversation loop:** The command uses `AskUserQuestion` in a multi-turn loop: scope questions → constraint questions → feature grouping → AC drafting → confirmation gate → write PRD. The agent handles all natural language reasoning. The command is the state machine; the agent is the reasoning engine.

**New gsd-tools subcommand needed: `list-prds`**

Returns a JSON array of all `.planning/PRD-*.md` and `.planning/PRD.md` files found in the project:

```javascript
// In gsd-tools.cjs, new case:
case 'list-prds': {
  const planDir = path.join(cwd, '.planning');
  let files = [];
  try {
    files = fs.readdirSync(planDir)
      .filter(f => f.startsWith('PRD') && f.endsWith('.md'))
      .map(f => path.join(planDir, f));
  } catch { /* no .planning dir */ }
  core.output(files, raw, files.join('\n'));
  break;
}
```

This is a 10-line addition to the existing `switch` block. No new module file needed.

---

### 2. Feature Map (`FEATURES.md`) Auto-Aggregation

**What it does:** Generates `.planning/FEATURES.md` — a structured overview aggregating features from all PRDs (`.planning/PRD*.md`) and from `@gsd-context`, `@gsd-api`, and `@gsd-todo` tags in code via `CODE-INVENTORY.md`.

**Stack decision: New `generate-feature-map` subcommand in `gsd-tools.cjs`. Zero new dependencies. Pure string processing using existing `arc-scanner.cjs` output.**

**How aggregation works:**

Step 1 — Collect PRD features: For each PRD file, extract feature names from `## Feature Groups` section (line scan, no AST). Each group becomes a FEATURES.md row.

Step 2 — Collect code signals: Read existing `CODE-INVENTORY.md` (already produced by `extract-tags`). Parse the Markdown table rows under `### @gsd-context` and `### @gsd-api` sections for descriptions — these represent implemented/planned features visible in code.

Step 3 — Merge and deduplicate: Simple string deduplication on feature name (case-insensitive prefix match). PRD features take precedence; code signals fill gaps for features without PRD coverage.

Step 4 — Write FEATURES.md.

**Why this is pure string processing (not LLM):**
- PRD feature groups are already structured (`Group: name — [AC-1, AC-2]`)
- `CODE-INVENTORY.md` is already a structured Markdown table (produced by `formatAsMarkdown` in `arc-scanner.cjs`)
- Deduplication on feature names is string comparison, not semantic reasoning
- An LLM call for aggregation would add latency and non-determinism to what is fundamentally a join operation

**New gsd-tools subcommand: `generate-feature-map`**

```javascript
case 'generate-feature-map': {
  const featureMap = require('./lib/feature-map.cjs');
  featureMap.cmdGenerateFeatureMap(cwd, args.slice(1));
  break;
}
```

**New CJS module: `get-shit-done/bin/lib/feature-map.cjs`**

This module (analogous to `arc-scanner.cjs` and `test-detector.cjs`) handles:
- `readPrdFeatures(planDir)` — scans PRD files for Feature Groups sections
- `readCodeFeatures(inventoryPath)` — parses CODE-INVENTORY.md table rows
- `mergeFeatures(prdFeatures, codeFeatures)` — deduplication logic
- `formatFeatureMap(features, projectName)` — produces FEATURES.md Markdown
- `cmdGenerateFeatureMap(cwd, args)` — CLI entry point

Module size estimate: ~150 lines. No external dependencies. Follows the same module shape as `arc-scanner.cjs`.

**FEATURES.md output format:**

```markdown
# FEATURES.md

**Generated:** [ISO timestamp]
**Project:** [name]
**PRD sources:** [count] files
**Code signals:** [count] tags

## Feature Overview

| Feature | Source | Status | ACs | Phase |
|---------|--------|--------|-----|-------|
| [name] | PRD: PRD-auth.md | draft | 3 | 1 |
| [name] | Code: @gsd-api | implemented | — | — |

## By Phase

### Phase 1
- [feature] ([AC count] ACs, source: PRD-auth.md)

### Unphased
- [feature] (code signal, file: src/auth.js:42)

## Coverage Gaps

Features in code without PRD coverage:
- [feature] — @gsd-context at src/foo.js:12, no PRD entry
```

**When to regenerate:** The `brainstorm` command auto-chains `generate-feature-map` after writing a new PRD (same pattern as `prototype` auto-chains `extract-tags`). The `extract-plan` command should also auto-chain it so code changes are reflected.

---

### 3. Architecture Mode for `/gsd:prototype`

**What it does:** Adds `--architecture` flag to `/gsd:prototype` that runs a skeleton-first pass — produces directory structure, interface/type definitions, and module boundaries BEFORE any implementation code. Useful for greenfield projects where the architecture needs to be established before prototyping individual features.

**Stack decision: Flag-based extension to existing `prototype.md` command + behavioral instruction addition to `gsd-prototyper.md`. Zero new files, zero new dependencies.**

**How it works — command layer:**

The `prototype.md` command gains a Step 0 flag check for `--architecture`. When present:
- Sets `architecture_mode = true`
- Passes `--architecture` flag in the Task() prompt to `gsd-prototyper`
- After `gsd-prototyper` completes, asks: "Architecture scaffold generated. Run implementation pass? [yes/no]" (unless `--non-interactive`)
- If yes: re-spawns `gsd-prototyper` WITHOUT `--architecture` (implementation pass uses the scaffold as input)

This two-pass approach is the correct model because:
- Architecture decisions (module boundaries, interfaces) should be validated before implementation fills them in
- The approval gate between passes prevents wasting implementation work on bad architecture
- `gsd-prototyper` already reads existing files before building (Step 1: `load_context`) — implementation pass naturally inherits the architecture scaffold

**How it works — agent layer:**

`gsd-prototyper.md` gains a new conditional section in its `plan_prototype` step:

```
If $ARGUMENTS contains --architecture:
  Architecture-mode build rules:
  - Create directory structure (empty __init__.py, index.ts, mod.rs, etc.)
  - Define interfaces, types, and abstract base classes — no implementations
  - Write module-level @gsd-context tags documenting each module's responsibility
  - Write @gsd-api tags for all public interfaces (no bodies, just signatures)
  - Write @gsd-decision tags for key architectural choices
  - DO NOT implement any function bodies
  - DO NOT write business logic
  - Output a ARCHITECTURE-NOTES.md file listing: modules created, interface contracts, dependency graph (text form)
```

**No new gsd-tools subcommand needed.** Architecture mode is entirely flag-driven behavior in the agent and command layers.

**Config key addition:** `arc.architecture_mode` — optional boolean stored in `.gsd-config.json`. Allows projects to set architecture mode as default for all prototype runs. Follows the `config.cjs` VALID_KEYS whitelist pattern — add `'arc.architecture_mode'` to the array.

---

## New Files Required (v1.2)

| File | Type | Purpose |
|------|------|---------|
| `commands/gsd/brainstorm.md` | Command (Markdown) | Orchestrates interactive conversation → PRD generation |
| `agents/gsd-brainstormer.md` | Agent (Markdown) | Handles PRD writing, AC extraction, feature grouping |
| `get-shit-done/bin/lib/feature-map.cjs` | CJS module | Aggregates PRD features + code signals into FEATURES.md |
| `commands/gsd/feature-map.md` | Command (Markdown) | Exposes `generate-feature-map` as `/gsd:feature-map` command |

**Modified files:**

| File | Change |
|------|--------|
| `commands/gsd/prototype.md` | Add `--architecture` flag handling (Steps 0 and post-completion) |
| `agents/gsd-prototyper.md` | Add architecture-mode build rules in `plan_prototype` step |
| `commands/gsd/extract-plan.md` | Auto-chain `generate-feature-map` after CODE-INVENTORY.md write |
| `get-shit-done/bin/gsd-tools.cjs` | Add `list-prds` and `generate-feature-map` cases to switch block |
| `get-shit-done/bin/lib/config.cjs` | Add `'arc.architecture_mode'` to VALID_KEYS array |

No new directories. All new files fit into existing `agents/` and `commands/gsd/` paths already in the `files` array of `package.json`. `feature-map.cjs` goes in `get-shit-done/bin/lib/` which is already included.

---

## What NOT to Add (v1.2 Specific)

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `gray-matter` / `front-matter` npm packages | PRD frontmatter keys (`status`, `phase`, `title`) are single-value strings resolvable with `line.split(':')` in a 10-line CJS function. Adding a YAML parser adds a runtime dep for a trivial parse. | Native string split in `feature-map.cjs` |
| `remark` / `unified` Markdown AST for PRD parsing | Feature Groups sections are fixed-format (`Group: name — [AC-1, AC-2]`). Line scanning is sufficient and faster. | `String.split('\n')` with prefix matching in `feature-map.cjs` |
| Separate `gsd-prd-formatter.md` agent | The brainstormer agent can write PRDs directly. A separate formatter agent adds a Task() spawn with no net benefit — one agent produces both the conversation and the output. | Inline PRD writing in `gsd-brainstormer.md` |
| LLM-driven feature map generation | Aggregation is a join operation (PRD features + code signals), not a reasoning task. Using an agent for this adds non-determinism and latency to what is pure string processing. | `feature-map.cjs` CJS module |
| New PRD file format (JSON, TOML, etc.) | Markdown with YAML frontmatter is already the format prototype.md expects for AC extraction (Step 2 uses `AC-N:` pattern matching). Changing format would break existing pipeline. | Extend existing `.planning/PRD.md` format |
| `/gsd:architecture` as a separate top-level command | Architecture mode is a variant of prototype behavior, not a different workflow. A flag keeps the command surface minimal and makes the two-pass flow explicit. | `--architecture` flag on `/gsd:prototype` |
| Persistent brainstorm session state in `.gsd-config.json` | Brainstorm is a one-shot conversation → PRD write. There is no need to resume a half-completed brainstorm session. The PRD is the durable artifact. | PRD file as the sole output |
| Remote PRD sources (Notion, Linear API) | Already explicitly out of scope in PROJECT.md. Local files only. | `--prd <path>` flag |

---

## Stack Patterns by Variant

**If project has multiple PRDs (complex multi-feature milestone):**
- Use `list-prds` to enumerate them
- Pass all paths to `generate-feature-map` for combined FEATURES.md
- Run `brainstorm` once per feature area, not once per project

**If project has no PRDs yet (greenfield):**
- Run `/gsd:brainstorm` first to produce `.planning/PRD.md`
- Then run `/gsd:prototype --prd .planning/PRD.md --architecture` for skeleton-first build
- Feature map auto-generates after prototype completes

**If project already has code with @gsd-tags but no PRDs:**
- Run `/gsd:feature-map` directly — aggregates from code signals only
- Coverage Gaps section in FEATURES.md shows which features need PRD backing
- Use gap list to drive retroactive `/gsd:brainstorm` sessions

---

## Integration Points (v1.2)

**`gsd-tools.cjs` additions:**

```
list-prds                 — returns JSON array of .planning/PRD*.md paths
generate-feature-map      — reads PRDs + CODE-INVENTORY.md, writes FEATURES.md
  --output <path>         — override default .planning/FEATURES.md
  --prd-dir <path>        — override default .planning/ for PRD discovery
```

**Auto-chain additions:**

| Command | After Which Step | What Chains |
|---------|-----------------|-------------|
| `brainstorm.md` | After PRD write | `generate-feature-map` |
| `extract-plan.md` | After CODE-INVENTORY.md write | `generate-feature-map` (if FEATURES.md exists) |
| `prototype.md` (architecture mode) | After scaffold pass | User confirmation gate, then implementation pass |

**Config additions:**

| Key | Type | Default | Purpose |
|-----|------|---------|---------|
| `arc.architecture_mode` | boolean | false | Make `--architecture` the default for all prototype runs |

Add to `VALID_KEYS` in `config.cjs`: `'arc.architecture_mode'`

---

## Version Compatibility (v1.2)

| Component | Version | Notes |
|-----------|---------|-------|
| `@anthropic-ai/claude-agent-sdk` | ^0.2.87 | Carry-forward from v1.1. No version bump needed for v1.2. |
| Node.js | >=20.0.0 | `fs.readdirSync`, `String.prototype.matchAll`, `RegExp` with `/gm` — all available in Node.js 20+ |
| `feature-map.cjs` | Internal | New module. Must pass `node:test` tests. Follows same zero-dep constraint as `arc-scanner.cjs`. |

---

## Confidence Assessment

| Area | Confidence | Basis |
|------|------------|-------|
| Zero-dep constraint maintained | HIGH | All v1.2 additions are Markdown files or CJS modules using built-ins only. No npm package needed for string parsing of fixed-format Markdown sections. |
| PRD frontmatter via line scan | HIGH | `status: draft`, `phase: 1` are single-key-value lines. `split(':')[1].trim()` is sufficient. Verified by reading PRD format used in `prototype.md` Step 2 extraction logic. |
| Feature map via CJS string processing | HIGH | `CODE-INVENTORY.md` is a fixed-format Markdown table produced by `formatAsMarkdown` in `arc-scanner.cjs` (read directly). PRD Feature Groups sections are fixed-format. No semantic ambiguity. |
| Architecture mode as flag extension | HIGH | `gsd-prototyper.md` already has conditional behavior based on `--phases` flag. Adding `--architecture` conditional follows identical pattern. Agent already reads existing files before building — two-pass works naturally. |
| `list-prds` as 10-line gsd-tools addition | HIGH | `readdirSync` with prefix/suffix filter is the same pattern already used in `scan-sessions` case. |
| `generate-slug` reuse for PRD filenames | HIGH | `generate-slug` is already in `gsd-tools.cjs` switch and confirmed working in `brainstorm`-adjacent workflows. |
| Two-pass architecture + implementation | MEDIUM | This is a new workflow pattern for this codebase. The mechanism (re-spawn agent with different flags) is proven by the existing iterate loop, but the two-pass UX has not been user-tested. Flag as needing validation in Phase 1 (brainstorm/architecture). |

---

## Sources

- Codebase audit (2026-03-29): `commands/gsd/prototype.md`, `agents/gsd-prototyper.md`, `get-shit-done/bin/lib/arc-scanner.cjs`, `get-shit-done/bin/lib/config.cjs`, `get-shit-done/bin/gsd-tools.cjs` (lines 919-960), `.planning/PROJECT.md` — read directly
- Previous research: `.planning/research/STACK.md` (v1.1) — carry-forward decisions confirmed
- `npm view @anthropic-ai/claude-agent-sdk version` → `0.2.87` (2026-03-29)
- Node.js >=20 built-in API docs — `fs.readdirSync`, `String.matchAll`, POSIX path handling — HIGH confidence (built-ins unchanged across 20/22/24)
- `.planning/config.json` (project) and `config.cjs` VALID_KEYS array — read directly for config extension pattern

---

*Stack research for: GSD Code-First Fork v1.2 — Brainstorm, Feature Map, Architecture Mode*
*Researched: 2026-03-29*
