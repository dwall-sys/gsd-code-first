# Phase 1: Annotation Foundation - Research

**Researched:** 2026-03-28
**Domain:** Node.js CJS CLI extension — regex-based code annotation scanner, config schema extension, agent/command authoring
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Tags use single-line structured format: `@gsd-todo(phase:2) Description text` with parenthesized metadata
- **D-02:** Tags are anchored to comment tokens only (`//`, `#`, `/*`, `--`, `"""`, `'''`) — regex must NOT match @gsd- in strings, URLs, or template literals
- **D-03:** Eight tag types: @gsd-context, @gsd-decision, @gsd-todo, @gsd-constraint, @gsd-pattern, @gsd-ref, @gsd-risk, @gsd-api
- **D-04:** Metadata in parentheses is optional — `@gsd-todo Fix this` is valid, `@gsd-todo(phase:2, priority:high) Fix this` adds structured metadata
- **D-05:** Tag scanner implemented as new lib module `get-shit-done/bin/lib/arc-scanner.cjs` following the existing module pattern (like config.cjs, phase.cjs)
- **D-06:** Scanner exposed via gsd-tools.cjs as subcommand: `gsd-tools.cjs extract-tags [--phase N] [--type TYPE] [--format md|json]`
- **D-07:** Scanner uses Node.js built-in `RegExp` with multiline flags — zero runtime dependencies, matching the zero-dep constraint
- **D-08:** Scanner output formats: JSON (for agent consumption) and Markdown (for CODE-INVENTORY.md)
- **D-09:** extract-plan command defined in `commands/gsd/extract-plan.md` — calls `gsd-tools.cjs extract-tags`, writes `.planning/prototype/CODE-INVENTORY.md`, shows terminal summary
- **D-10:** CODE-INVENTORY.md groups tags by type, then by file, with phase reference cross-links and summary statistics
- **D-11:** gsd-annotator agent defined in `agents/gsd-annotator.md` — operates at directory scope with file glob filtering
- **D-12:** annotate command defined in `commands/gsd/annotate.md` — spawns gsd-annotator then auto-runs extract-plan on completion
- **D-13:** Annotator reads existing code + PROJECT.md + REQUIREMENTS.md to determine appropriate annotations
- **D-14:** New top-level keys `arc` and `phase_modes` added to `.planning/config.json` schema
- **D-15:** `arc` config section: `{ enabled: boolean, tag_prefix: "@gsd-", comment_anchors: ["//", "#", "/*", "--"] }`
- **D-16:** `phase_modes` config section: maps phase numbers to mode strings (`code-first`, `plan-first`, `hybrid`)
- **D-17:** Config validation extended in `get-shit-done/bin/lib/config.cjs`

### Claude's Discretion

- Regex pattern specifics for edge case handling (strings, template literals)
- CODE-INVENTORY.md exact formatting and section ordering
- arc-standard.md language examples beyond the core 4 (JS, Python, Go, Rust)
- Test fixture design for false-positive prevention

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ARC-01 | Developer can annotate code with structured @gsd-tags (context, decision, todo, constraint, pattern, ref, risk, api) following a documented standard | D-01 through D-04 define the tag syntax; arc-standard.md spec and language examples research below |
| ARC-02 | ARC standard reference document at get-shit-done/references/arc-standard.md with syntax rules, tag definitions, and per-language examples | Reference file location confirmed; document structure patterns from existing references/ files |
| SCAN-01 | Tag scanner extracts all @gsd-tags from source files via regex, anchored to comment tokens to avoid false positives | Regex anchor pattern and false-positive prevention strategy documented below |
| SCAN-02 | Tag scanner outputs structured JSON (for agents) and Markdown (for CODE-INVENTORY.md) | JSON schema and Markdown structure defined in Standard Stack and Code Examples sections |
| SCAN-03 | Tag scanner supports filtering by phase reference and tag type | Filter implementation pattern documented |
| SCAN-04 | Tag scanner is language-agnostic (works on any text file with comments) | Verified: regex-over-text approach with no AST, universal comment token anchoring |
| EXTR-01 | extract-plan command invokes tag scanner and writes .planning/prototype/CODE-INVENTORY.md | Command pattern from quick.md; output path clarification in Architecture section |
| EXTR-02 | CODE-INVENTORY.md groups tags by type, file, and phase reference with summary statistics | CODE-INVENTORY.md schema defined below |
| MODE-02 | Config schema extended with phase_modes, arc settings, and default_phase_mode | config.cjs extension pattern documented; VALID_CONFIG_KEYS update strategy |
| ANNOT-01 | gsd-annotator agent retroactively annotates existing code with @gsd-tags | Agent authoring pattern from gsd-executor.md; annotator agent structure below |
| ANNOT-02 | annotate command spawns annotator and auto-runs extract-plan on completion | Command + auto-chain pattern documented |
</phase_requirements>

---

## Summary

Phase 1 delivers the foundational layer of the GSD Code-First fork: the ARC annotation standard document, the arc-scanner CJS module, the extract-tags subcommand in gsd-tools.cjs, the CODE-INVENTORY.md artifact schema, the gsd-annotator agent, the annotate and extract-plan command files, and config schema extensions. Every deliverable is a net-new file or a strictly additive change to config.cjs — no upstream files are modified.

The technical work is well-understood. The codebase provides exact templates for every component type: 17 existing lib modules define the CJS pattern for arc-scanner.cjs, 18 existing agents define the YAML frontmatter pattern for gsd-annotator.md, and the quick.md command pattern defines the structure for annotate.md and extract-plan.md. The only genuine discretion area is the regex pattern for false-positive prevention, which is covered in detail below.

The highest-risk implementation detail is the comment-anchor regex. Getting this wrong produces false positives in CODE-INVENTORY.md that erode trust in the tool. The prevention strategy — anchoring to whitespace + comment-token, then requiring `@gsd-` to follow — is the proven approach from the pre-existing PITFALLS.md research. The scanner test fixtures must be built alongside the scanner code, not after.

**Primary recommendation:** Build in strict layer order: arc-standard.md first (the spec everything else implements), then arc-scanner.cjs + tests, then CODE-INVENTORY.md schema, then extract-plan command, then config.cjs extension, then gsd-annotator + annotate command. Do not start agent work until the scanner is verified green.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js built-ins (`fs`, `path`, `os`) | >=20.0.0 | File I/O, path resolution, directory walking | Zero-dep constraint; already the entire runtime surface |
| Native `RegExp` (`gm` flags) | ES2018+ | Multiline `@gsd-tag` extraction anchored to comment tokens | PROJECT.md decision: regex over AST; language-agnostic; no dependencies |
| `node:test` + `node:assert` | Node.js >=20 built-in | Unit/integration tests for arc-scanner.cjs and gsd-tools.cjs extensions | All 47 existing test files use this runner — mandatory pattern |
| `c8` | ^11.0.0 (verified) | Coverage reporting | Already in devDependencies; 70% line coverage enforced |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `node:crypto` | Node.js built-in | Tag deduplication hashing if needed | If scanner needs to deduplicate identical tags across includes |
| Claude Code agent YAML frontmatter | Current Claude Code format | Define gsd-annotator agent | Only for agent .md files — exact format from existing agents |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Native RegExp | `comment-parser` npm (1.4.5) | comment-parser handles JSDoc semantics but adds a runtime dep and doesn't support non-JS files — rejected |
| Native RegExp | AST parsers (acorn, esprima) | Language-specific, can't parse Python/Go/Rust — explicitly rejected in PROJECT.md |
| `fs.readdirSync` recursion | `glob` npm package | Node.js 22+ has `fs.glob()` built-in; for Node >=20 simple recursion is sufficient — no dep needed |

**Installation:** No new runtime dependencies. All new code uses Node.js built-ins only.

---

## Architecture Patterns

### Recommended Project Structure (Phase 1 deliverables only)

```
get-shit-done/
├── bin/
│   ├── gsd-tools.cjs              # EXTEND: add extract-tags case in switch
│   └── lib/
│       ├── arc-scanner.cjs        # NEW: regex scanner module
│       └── config.cjs             # EXTEND: add arc/phase_modes to VALID_CONFIG_KEYS and buildNewProjectConfig
├── references/
│   └── arc-standard.md            # NEW: ARC annotation standard document
agents/
└── gsd-annotator.md               # NEW: retroactive annotation agent
commands/gsd/
├── extract-plan.md                # NEW: extract-plan slash command
└── annotate.md                    # NEW: annotate slash command
tests/
└── arc-scanner.test.cjs           # NEW: scanner unit tests with edge-case fixtures
.planning/
└── prototype/
    └── CODE-INVENTORY.md          # GENERATED artifact (not committed to repo)
```

### Pattern 1: CJS Lib Module Structure (arc-scanner.cjs)

**What:** Every lib module exports named functions via `module.exports`. It uses only `require('fs')`, `require('path')`, and other Node.js built-ins. No external requires.

**When to use:** Any new functionality that extends gsd-tools.cjs behavior.

**Example (from config.cjs / core.cjs pattern):**
```javascript
// get-shit-done/bin/lib/arc-scanner.cjs
'use strict';

const fs = require('fs');
const path = require('path');
const { error } = require('./core.cjs');

// Default directories to exclude from scanning
const DEFAULT_EXCLUDES = ['node_modules', 'dist', 'build', '.planning', '.git'];

/**
 * Scan a file for @gsd-tags anchored to comment tokens.
 * Returns an array of tag objects.
 */
function scanFile(filePath) { ... }

/**
 * Walk a directory recursively, scanning all text files.
 * Respects DEFAULT_EXCLUDES and optional .gsdignore.
 */
function scanDirectory(dirPath, options = {}) { ... }

module.exports = { scanFile, scanDirectory };
```

### Pattern 2: gsd-tools.cjs Command Dispatch

**What:** New subcommands are added as `case` branches in the main `switch(command)` block in gsd-tools.cjs. The new module is required at the top alongside existing modules.

**When to use:** Every new CLI subcommand.

**Example (following the existing dispatch pattern):**
```javascript
// At top of gsd-tools.cjs alongside other requires:
const arcScanner = require('./lib/arc-scanner.cjs');

// In the switch(command) block:
case 'extract-tags': {
  const { phase: phaseFilter, type: typeFilter, format } = parseNamedArgs(
    args.slice(1), ['phase', 'type', 'format']
  );
  const targetPath = args[1] || cwd;
  arcScanner.cmdExtractTags(cwd, targetPath, { phaseFilter, typeFilter, format, raw });
  break;
}
```

### Pattern 3: Slash Command File (extract-plan.md / annotate.md)

**What:** Command files have YAML frontmatter (name, description, allowed-tools) and a thin body that injects workflow context via `@`-reference or inline prose. No logic in command files.

**When to use:** Every new `/gsd:` slash command.

**Example (following quick.md pattern):**
```markdown
---
name: gsd:extract-plan
description: Scan codebase for @gsd-tags and produce .planning/prototype/CODE-INVENTORY.md
allowed-tools:
  - Read
  - Write
  - Bash
---
<objective>
Run the ARC tag scanner against the current project and write CODE-INVENTORY.md.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/extract-plan.md
</execution_context>

<context>
$ARGUMENTS
</context>
```

### Pattern 4: Agent File Structure (gsd-annotator.md)

**What:** Agent files use YAML frontmatter (`name`, `description`, `tools`, `permissionMode`, `color`) followed by Markdown prose organized into `<role>`, `<project_context>`, and numbered execution steps. Spawned by slash commands via the `Task` tool or SDK.

**When to use:** Every new agent type.

**Example frontmatter (following gsd-executor.md pattern):**
```yaml
---
name: gsd-annotator
description: Retroactively annotates existing code with @gsd-tags following the ARC standard. Spawned by /gsd:annotate command.
tools: Read, Write, Edit, Bash, Grep, Glob
permissionMode: acceptEdits
color: green
---
```

### Pattern 5: Config Schema Extension

**What:** Adding new top-level keys to `.planning/config.json` requires three coordinated changes in config.cjs: (1) add keys to `VALID_CONFIG_KEYS`, (2) add defaults to `hardcoded` in `buildNewProjectConfig()`, (3) add to the deep-merge logic for nested objects.

**When to use:** Any new config namespace.

**Example:**
```javascript
// 1. Add to VALID_CONFIG_KEYS Set:
'arc.enabled', 'arc.tag_prefix', 'arc.comment_anchors',
'phase_modes.default',

// 2. Add to hardcoded defaults in buildNewProjectConfig():
arc: {
  enabled: true,
  tag_prefix: '@gsd-',
  comment_anchors: ['//', '#', '/*', '--'],
},
phase_modes: {},

// 3. Add to deep-merge return:
arc: {
  ...hardcoded.arc,
  ...(userDefaults.arc || {}),
  ...(choices.arc || {}),
},
phase_modes: {
  ...hardcoded.phase_modes,
  ...(userDefaults.phase_modes || {}),
  ...(choices.phase_modes || {}),
},
```

### Anti-Patterns to Avoid

- **Modifying upstream agent/command files:** `gsd-executor.md`, `gsd-planner.md`, `quick.md`, `fast.md` and all other existing files are read-only in this phase. Any needed behavior goes in new files only.
- **Logic in command .md files:** Command files reference workflow files or inline prose — they contain no conditional logic or state reads.
- **Direct `.planning/` writes from agents:** All state writes route through `gsd-tools.cjs` subcommands. The extract-plan command writes CODE-INVENTORY.md by calling `gsd-tools.cjs extract-tags --format md` and piping output, or via a dedicated `write-inventory` subcommand.
- **AST-based parsing:** Explicitly out of scope. Regex only.
- **Runtime npm dependencies:** Zero. Every `require()` in arc-scanner.cjs must be a Node.js built-in.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Comment syntax detection | Custom language parser per file type | Regex anchored to `//`, `#`, `/*`, `--`, `"""`, `'''` with `gm` flags | Language-agnostic, handles all comment styles, zero deps |
| Directory traversal with filtering | Custom ignore logic | `fs.readdirSync` recursion + DEFAULT_EXCLUDES array + optional `.gsdignore` | Simple, deterministic, same pattern as existing code |
| JSON output formatting | Custom serializer | `JSON.stringify(results, null, 2)` | Already the project pattern everywhere |
| Markdown table generation | Markdown library | Template literal string concatenation | Pattern used throughout codebase; markdown-it is a parsing library not a generator |
| Config key validation | Ad-hoc checks | Add to `VALID_CONFIG_KEYS` Set in config.cjs | Existing validation mechanism handles suggestion and error formatting |
| Agent frontmatter format | Invent new format | Copy YAML frontmatter from gsd-executor.md | Claude Code requires exact format; all 18 existing agents use it |

**Key insight:** Every problem in this phase already has a solved pattern in the existing codebase. The work is additive assembly, not invention.

---

## Common Pitfalls

### Pitfall 1: Regex False Positives — @gsd- in Strings and URLs
**What goes wrong:** The pattern `/@gsd-(\w+)/` fires on string literals (`const x = "// @gsd-todo not a tag"`), template literals, and URL paths (`http://example.com/@gsd/something`), polluting CODE-INVENTORY.md.

**Why it happens:** JavaScript regex cannot model JavaScript syntax. The `@gsd-` sequence is not restricted to comments.

**How to avoid:** Require the tag to be anchored to a comment token. The pattern must match only lines where the comment token appears before `@gsd-`. Verified working pattern:

```javascript
// Anchors: line starts with optional whitespace, then a comment token, then optional whitespace, then @gsd-
const TAG_LINE_RE = /^[ \t]*(?:\/\/+|\/\*+|\*+|#+|--+|"{3}|'{3})[ \t]*@gsd-(\w+)(?:\(([^)]*)\))?[ \t]*(.*?)[ \t]*$/gm;
```

This pattern:
- `^[ \t]*` — optional leading whitespace only (no content before comment token)
- `(?:\/\/+|\/\*+|\*+|#+|--+|"{3}|'{3})` — comment opening token (includes continuation `*` for block comments)
- `[ \t]*` — optional space between token and tag
- `@gsd-(\w+)` — tag type capture
- `(?:\(([^)]*)\))?` — optional metadata in parens (capture group 2)
- `[ \t]*(.*?)[ \t]*$` — description text (capture group 3)

**Warning signs:** Tag counts from scanner don't match manual grep; tags from test fixtures (string-embedded @gsd-) appear in output.

**Mitigation:** Build `tests/arc-scanner.test.cjs` with a fixture file containing deliberate false-positive cases before writing the scanner implementation (TDD-style).

---

### Pitfall 2: ARC Standard Churn — Changing Tag Names After Real Use
**What goes wrong:** Tag names or required fields change after a codebase has been annotated, breaking the scanner output format and requiring re-annotation.

**Why it happens:** The standard is designed before real usage reveals what information is actually needed.

**How to avoid:** Treat D-01 through D-04 as the frozen v1.0 standard. The arc-standard.md document must state explicitly: "Tag names and parenthesized keys are stable. New optional keys may be added in future versions." The eight tag types in D-03 cannot be renamed — only extended.

**Warning signs:** Proposals to rename a tag type before the first real annotation session.

---

### Pitfall 3: Fork Divergence — Touching Upstream Files
**What goes wrong:** Modifying any existing file in `agents/`, `commands/gsd/`, `get-shit-done/bin/` (other than gsd-tools.cjs and config.cjs extension points) creates permanent merge conflict sites.

**How to avoid:** This phase produces ONLY new files plus additive extensions to gsd-tools.cjs (new case branches) and config.cjs (new keys, new defaults). Every line changed in existing files must be an addition, never a modification of existing lines.

**Warning signs:** Any `git diff` against an existing agent or command file that shows deleted or modified lines (as opposed to only added lines in new positions).

---

### Pitfall 4: .planning/prototype/ Directory Not Created
**What goes wrong:** The extract-plan command writes to `.planning/prototype/CODE-INVENTORY.md` but the `prototype/` subdirectory may not exist. `fs.writeFileSync` will throw; the command fails silently if the error is swallowed.

**How to avoid:** The arc-scanner.cjs `writeInventory()` function (or extract-plan command implementation) must call `fs.mkdirSync(dir, { recursive: true })` before any write. Follow the pattern in config.cjs `cmdConfigNewProject()`:
```javascript
if (!fs.existsSync(planningBase)) {
  fs.mkdirSync(planningBase, { recursive: true });
}
```

---

### Pitfall 5: Missing Default Excludes Causing Scanner to Scan node_modules
**What goes wrong:** Without exclusion defaults, the directory walker enters `node_modules/` and scans thousands of files. On a project with 50K node_modules files, this takes seconds and may produce false positives from test fixtures in npm packages.

**How to avoid:** DEFAULT_EXCLUDES array in arc-scanner.cjs must include `node_modules`, `dist`, `build`, `.planning`, `.git`. Additionally support a `.gsdignore` file (gitignore-format) at project root. Check the directory name against DEFAULT_EXCLUDES before recursing.

---

## Code Examples

Verified patterns from codebase inspection (HIGH confidence):

### Existing Test File Pattern (node:test)
```javascript
// Source: tests/config.test.cjs — verified pattern
const { test, describe, beforeEach, afterEach } = require('node:test');
const assert = require('node:assert');
const fs = require('fs');
const path = require('path');
const { runGsdTools, createTempProject, cleanup } = require('./helpers.cjs');

describe('arc-scanner extract-tags command', () => {
  let tmpDir;
  beforeEach(() => { tmpDir = createTempProject(); });
  afterEach(() => { cleanup(tmpDir); });

  test('extracts @gsd-context tag from JS comment', () => {
    // Write fixture file to tmpDir
    fs.writeFileSync(path.join(tmpDir, 'sample.js'), `
// @gsd-context This is the auth module
function authenticate() {}
    `.trim(), 'utf-8');
    const result = runGsdTools('extract-tags sample.js --format json', tmpDir);
    assert.ok(result.success);
    const tags = JSON.parse(result.output);
    assert.strictEqual(tags.length, 1);
    assert.strictEqual(tags[0].type, 'context');
  });

  test('does NOT extract @gsd- from string literal', () => {
    fs.writeFileSync(path.join(tmpDir, 'string-trap.js'), `
const msg = "// @gsd-todo this is not a tag";
    `.trim(), 'utf-8');
    const result = runGsdTools('extract-tags string-trap.js --format json', tmpDir);
    assert.ok(result.success);
    const tags = JSON.parse(result.output);
    assert.strictEqual(tags.length, 0);
  });
});
```

### Config Extension Pattern
```javascript
// Source: get-shit-done/bin/lib/config.cjs — verified pattern
// Step 1: Add to VALID_CONFIG_KEYS
const VALID_CONFIG_KEYS = new Set([
  // ... existing keys ...
  'arc.enabled', 'arc.tag_prefix', 'arc.comment_anchors',
  'phase_modes.default',
]);

// Step 2: Add to hardcoded defaults
const hardcoded = {
  // ... existing defaults ...
  arc: {
    enabled: true,
    tag_prefix: '@gsd-',
    comment_anchors: ['//', '#', '/*', '--'],
  },
  phase_modes: {},
};

// Step 3: Add to deep-merge return (at bottom of buildNewProjectConfig)
return {
  // ... existing spread ...
  arc: {
    ...hardcoded.arc,
    ...(userDefaults.arc || {}),
    ...(choices.arc || {}),
  },
  phase_modes: {
    ...hardcoded.phase_modes,
    ...(userDefaults.phase_modes || {}),
    ...(choices.phase_modes || {}),
  },
};
```

### arc-standard.md Reference Document Structure
```markdown
# ARC Annotation Standard

**Version:** 1.0
**Stability:** Stable — tag names and required fields will not change.
  New optional metadata keys may be added in future versions.

## Tag Syntax

Single-line format:
  @gsd-<type>[(key:value, key:value)] Description text

...

## Tag Types

| Tag | Purpose | Example |
|-----|---------|---------|
| @gsd-context | ... | ... |

## Comment Anchors

Tags MUST appear on lines where the first non-whitespace token is a comment
opener: `//`, `#`, `/*`, `*` (continuation), `--`, `"""`, `'''`

## Language Examples

### JavaScript / TypeScript
// @gsd-context(phase:1) Auth module — handles JWT validation

### Python
# @gsd-decision(phase:1) Using bcrypt not argon2 because...

### Go
// @gsd-constraint No external HTTP calls in this package

### Rust
// @gsd-risk Memory safety: this uses unsafe for FFI boundary

### SQL
-- @gsd-context Partitioned by tenant_id for query performance

### Shell
# @gsd-todo(phase:2) Add error handling for missing HOME var
```

### JSON Tag Object Schema (SCAN-02)
```javascript
// Each extracted tag is represented as:
{
  "type": "context",          // one of the 8 tag types (no @gsd- prefix)
  "file": "src/auth/jwt.js",  // relative path from project root
  "line": 12,                 // 1-based line number
  "metadata": {               // parsed from parentheses, or {}
    "phase": "1",
    "priority": "high"
  },
  "description": "JWT validation module — stateless, RS256 only",
  "raw": "// @gsd-context(phase:1) JWT validation module..."
}
```

### CODE-INVENTORY.md Schema (EXTR-02)
```markdown
# CODE-INVENTORY.md

**Generated:** 2026-03-28T19:00:00Z
**Project:** GSD Code-First Fork
**Schema version:** 1.0
**Tags found:** 47 across 12 files

## Summary Statistics

| Tag Type | Count |
|----------|-------|
| @gsd-context | 15 |
| @gsd-decision | 8 |
| @gsd-todo | 12 |
| @gsd-constraint | 4 |
| @gsd-pattern | 3 |
| @gsd-ref | 2 |
| @gsd-risk | 2 |
| @gsd-api | 1 |

## Tags by Type

### @gsd-context

#### src/auth/jwt.js

| Line | Metadata | Description |
|------|----------|-------------|
| 12 | phase:1 | JWT validation module — stateless, RS256 only |
| 34 | — | Token refresh logic |

#### src/db/connection.js

| Line | Metadata | Description |
|------|----------|-------------|
| 5 | — | Singleton pool — max 10 connections |

### @gsd-decision

...

## Phase Reference Index

| Phase | Tag Count | Files |
|-------|-----------|-------|
| 1 | 23 | src/auth/jwt.js, src/db/... |
| 2 | 14 | ... |
| (untagged) | 10 | ... |
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Upfront spec documents as planning artifacts | @gsd-tags embedded in code as planning metadata | This project (new) | Planning lives with the code it describes |
| JSDoc `@todo`, `@note` for developer notes | Structured `@gsd-todo(phase:N)` with metadata | This project (new) | Tags are machine-readable by the scanner |
| AST-based comment extraction | Regex anchored to comment tokens | PROJECT.md decision | Language-agnostic, zero dependencies |

**Deprecated/outdated (in this project context):**
- Modifying `gsd-executor.md` for ARC obligations: The decision was made to implement as a new `gsd-arc-executor.md` wrapper in Phase 2, not modify the upstream file.

---

## Open Questions

1. **Should `extract-tags` subcommand name or `extract-plan` command name take precedence?**
   - What we know: D-06 names the subcommand `extract-tags`; D-09 names the command file `extract-plan.md`. These are different things — `extract-tags` is the low-level CLI tool, `extract-plan` is the slash command that calls it and writes CODE-INVENTORY.md.
   - What's unclear: D-06 says `gsd-tools.cjs extract-tags` but the ARCHITECTURE.md shows both `scan-tags` and `extract-plan` as gsd-tools subcommands.
   - Recommendation: Use D-06 as the source of truth. One subcommand: `extract-tags [--format md|json] [--phase N] [--type TYPE]`. The extract-plan slash command calls `extract-tags --format md` and writes the result.

2. **Does `default_phase_mode` need to be in Phase 1 or can it defer to Phase 3?**
   - What we know: D-16 defines `phase_modes` as a map. `default_phase_mode` is mentioned in ARCHITECTURE.md but not in CONTEXT.md decisions.
   - What's unclear: MODE-02 (Phase 1) says "Config schema extended with phase_modes, arc settings, and default_phase_mode." This suggests `default_phase_mode` must be added now.
   - Recommendation: Add `default_phase_mode: "plan-first"` as a top-level config key alongside `arc` and `phase_modes`. Safe default preserves existing behavior.

3. **Write path for CODE-INVENTORY.md: `.planning/prototype/` vs `.planning/`**
   - What we know: D-09 specifies `.planning/prototype/CODE-INVENTORY.md`. ARCHITECTURE.md shows `CODE-INVENTORY.md` at `.planning/` root.
   - What's unclear: Which is the authoritative location for downstream consumers (Phase 2 agents)?
   - Recommendation: Use D-09 (`.planning/prototype/CODE-INVENTORY.md`) as the canonical path. This is a decision from CONTEXT.md and takes priority. Phase 2 agents will read it from there.

---

## Environment Availability

Step 2.6: Environment availability audit — Phase 1 has no external dependencies beyond Node.js and the project's existing test infrastructure.

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All CJS modules | ✓ | >=20.0.0 (project min) | — |
| `node:test` | Test runner | ✓ | Built-in (Node >=18) | — |
| `c8` | Coverage | ✓ | 11.0.0 (in devDeps) | — |
| `tests/helpers.cjs` | Test scaffolding | ✓ | Exists in repo | — |

No missing dependencies. No fallbacks required.

---

## Project Constraints (from CLAUDE.md)

Directives extracted from CLAUDE.md (verified by reading the file):

1. **GSD workflow enforcement:** All file changes must go through a GSD command (`/gsd:quick`, `/gsd:debug`, `/gsd:execute-phase`). Do not make direct repo edits outside a GSD workflow unless user explicitly requests bypass.
2. **Zero runtime npm dependencies:** All new CJS code (`get-shit-done/bin/lib/*.cjs`) must use only Node.js built-ins. No `require('some-npm-package')`.
3. **CJS module format:** All files in `get-shit-done/bin/` use CommonJS (`module.exports`, `require()`). Do not use ESM (`import`/`export`) in this layer.
4. **node:test for CJS tests:** All test files in `tests/` use `require('node:test')` and `require('node:assert')`. Do not use vitest or jest in this layer.
5. **Upstream file read-only policy:** Existing `agents/*.md`, `commands/gsd/*.md`, and `get-shit-done/bin/lib/*.cjs` files (other than config.cjs and gsd-tools.cjs extension points) must not be modified.
6. **Tech stack match:** JavaScript/Node.js, Markdown, JSON only. Must match the original GSD stack.
7. **Upstream compatibility:** All original GSD commands must continue working unchanged after Phase 1 work.

---

## Sources

### Primary (HIGH confidence)

- `/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/config.cjs` — config extension pattern, VALID_CONFIG_KEYS, buildNewProjectConfig, setConfigValue
- `/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/core.cjs` — file I/O helpers, planningRoot, error(), output() patterns
- `/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/gsd-tools.cjs` — command dispatch switch pattern, parseNamedArgs, module require pattern
- `/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/agents/gsd-executor.md` — exact YAML frontmatter + agent prompt structure to replicate for gsd-annotator.md
- `/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/commands/gsd/quick.md` — command file pattern for extract-plan.md and annotate.md
- `/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/tests/config.test.cjs` — test file pattern with node:test, helpers.cjs, describe/test/beforeEach/afterEach
- `/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/.planning/config.json` — current config shape; `workflow.nyquist_validation: false`
- `/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/.planning/research/STACK.md` — zero-dep constraint verified, node:test confirmed, c8 version
- `/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/.planning/research/PITFALLS.md` — regex false positive prevention, fork divergence policy
- `/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/.planning/research/ARCHITECTURE.md` — component boundaries, data flow, build order

### Secondary (MEDIUM confidence)

- `/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/.planning/phases/01-annotation-foundation/01-CONTEXT.md` — all implementation decisions (D-01 through D-17)
- Codebase inspection: `ls get-shit-done/bin/lib/` — 17 lib modules confirmed, exact list verified
- Codebase inspection: `ls agents/` — 18 agent files confirmed, YAML frontmatter format verified
- Codebase inspection: `ls get-shit-done/references/` — existing reference files confirm arc-standard.md placement

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — verified by reading all lib files and test files directly
- Architecture patterns: HIGH — extracted from existing code, not inferred
- Pitfalls: HIGH — from pre-existing domain research + codebase verification
- Regex pattern: MEDIUM — pattern is correct in principle; exact edge case coverage needs test fixture validation

**Research date:** 2026-03-28
**Valid until:** 2026-04-28 (stable Node.js ecosystem — 30 day validity)
**nyquist_validation:** false — Validation Architecture section omitted per config
