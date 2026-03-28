# Technology Stack

**Project:** GSD Code-First Fork
**Researched:** 2026-03-28
**Scope:** Additions to existing Node.js CLI framework for code annotation parsing, tag extraction, AI agent workflows, and npm distribution

---

## Ruling Constraint

The existing codebase is pure Node.js CommonJS (`.cjs`) with **zero runtime npm dependencies**. Every lib file in `get-shit-done/bin/lib/` requires only Node.js built-ins (`fs`, `path`, `child_process`, `os`, `readline`, `crypto`, `vm`). All new additions for the Code-First fork MUST maintain this constraint. Adding runtime dependencies for the annotation scanner would introduce install-time friction, increase surface area for supply chain attacks, and violate the project's established zero-dep pattern.

---

## Recommended Stack

### Runtime Layer (gsd-tools.cjs extensions)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Node.js built-ins | >=20.0.0 (project minimum) | All I/O, path resolution, regex | Already the entire runtime dependency surface. No additions needed. |
| Native `RegExp` with `s` (dotAll) flag | ES2018, Node.js >=10 | Multiline `@gsd-tag` extraction from comment blocks | `[\s\S]*?` or `/s` flag handles multiline block comments. Language-agnostic, no parser needed. Pure regex is sufficient for structured `@gsd-` prefixed annotations per PROJECT.md decision. |
| `fs.readdirSync` + recursive walk | Node.js built-in | Source file discovery across project tree | No glob library needed. Simple recursive readdir with extension filtering handles the tag scanner use case. |
| `node:crypto` | Node.js built-in | Deduplication hashing for tags (if needed) | Already imported in install.js; zero cost to use. |

**Decision rationale for regex over AST:** PROJECT.md explicitly records "Regex-based tag extraction — Simpler than AST parsing, language-agnostic, sufficient for structured comments." Confirmed correct. AST parsers (acorn, esprima, babel) are language-specific, add dependencies, and cannot parse non-JS files (Python, Ruby, Rust). `@gsd-tags` appear in any comment syntax (single-line `//`, block `/* */`, hash `#`). Regex with `gm` and `s` flags handles all cases.

### Agent Layer (new `.md` agent files)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Claude Code agent frontmatter (YAML) | Current Claude Code format | Define new agents (`gsd-prototyper`, `gsd-code-planner`, `gsd-annotator`) | All 19 existing agents use this exact format. No alternative. |
| Markdown prose | — | Agent system prompts, tool instructions, workflow steps | Existing format. New agents follow identical structure to `gsd-executor.md`, `gsd-planner.md`. |

### SDK Layer (TypeScript — `sdk/` subdirectory only)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| TypeScript | ^5.7.0 (current in sdk/package.json) | SDK type safety | Already in use in `sdk/`. New SDK work stays TypeScript. The gsd-tools.cjs extensions are NOT TypeScript — they go in the CJS layer. |
| `@anthropic-ai/claude-agent-sdk` | ^0.2.86 (latest as of 2026-03-28) | Programmatic agent invocation for `prototype`, `iterate`, `annotate` commands that need to spawn agents | Already the SDK dependency (`sdk/package.json` pins `^0.2.84`). Use when commands need to spawn an agent programmatically rather than printing a slash-command. |
| Node.js ESM (`node:fs/promises`, `node:path`) | Node.js >=20 | SDK async I/O | SDK already uses ESM. Stays consistent. |

**Boundary rule:** `get-shit-done/bin/` and the new tag scanner — CJS, zero deps. `sdk/src/` — TypeScript ESM, deps allowed. Do not blur this boundary.

### Build & Distribution

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| esbuild | ^0.27.4 (current npm) | Bundles hooks for distribution (`hooks/dist/`) | Already in devDependencies, used by `scripts/build-hooks.js`. The same `prepublishOnly` pattern can copy/validate new hook files. No change to build approach. |
| `scripts/build-hooks.js` pattern | Existing | Pre-publish validation via `vm.Script` syntax check | Already prevents shipping broken hooks. New hook files follow the same copy-and-validate flow. |
| npm `files` array in package.json | Current npm | Whitelist distribution files | Already configured: `["bin", "commands", "get-shit-done", "agents", "hooks/dist", "scripts"]`. New agent `.md` files go in `agents/`, new commands in `commands/`. No change needed to the `files` array unless new directories are added. |
| `npx gsd-code-first@latest` | npm registry | End-user install pattern | Already the target. `package.json` `name` is `gsd-code-first`. The `bin` entry (`get-shit-done-cc`) routes to `bin/install.js`. No change. |

### Testing

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `node:test` (built-in) | Node.js >=20 | Unit tests for all `.cjs` code | All 47 existing test files (`tests/*.test.cjs`) use `require('node:test')` and `require('node:assert')`. The tag scanner and new gsd-tools commands MUST follow this pattern — not vitest. |
| `c8` | ^11.0.0 (current npm) | Coverage reporting for CJS layer | Already in devDependencies. Used in `test:coverage` script. Project enforces 70% line coverage. |
| vitest | ^4.1.2 (latest as of 2026-03-28) | SDK TypeScript tests only (`sdk/src/**/*.test.ts`) | SDK uses vitest via its own `sdk/package.json`. Do NOT use vitest for CJS tests — the project actively avoids this. `vitest.config.ts` in root scopes only to `sdk/`. |

---

## What NOT to Use

| Rejected Option | Why Rejected |
|-----------------|-------------|
| `comment-parser` npm package (1.4.5, 2.3M weekly downloads) | Adds a runtime dependency. The existing codebase has zero. The package handles JSDoc semantics (types, optional params) that `@gsd-tags` don't need. Native regex handles the use case fully. |
| `commander` / `yargs` / `oclif` | The CLI already implements its own arg parsing via `parseNamedArgs()` in `gsd-tools.cjs`. Adding a CLI framework mid-project would require migrating all existing commands and violates the zero-dep constraint. |
| AST parsers (`acorn`, `esprima`, `@babel/parser`) | Language-specific. Can't parse Python, Ruby, or shell annotations. PROJECT.md explicitly chose regex over AST. |
| `mastra` / `kaibanjs` / AI orchestration frameworks | GSD agents are orchestrated via Claude Code's native agent spawning mechanism (slash commands, `Task` tool, SDK). These frameworks solve a different problem and add massive dependency trees. |
| `glob` npm package | Node.js 22+ has `fs.glob()` built-in. For Node.js 20, `readdirSync` with recursion is sufficient for the tag scanner use case. |
| Jest | SDK already uses vitest. CJS tests already use node:test. Introducing a third test runner adds config complexity with no benefit. |
| `marked` / `markdown-it` | CODE-INVENTORY.md is generated by string concatenation / template literals, not by parsing markdown. These are parsing libraries, not generation tools. No markdown parsing is required. |

---

## Installation

No new runtime dependencies are added. The extensions live in:

```
get-shit-done/bin/lib/tag-scanner.cjs     (new CJS module)
get-shit-done/bin/gsd-tools.cjs           (extended with new commands)
agents/gsd-prototyper.md                  (new agent)
agents/gsd-code-planner.md                (new agent)
agents/gsd-annotator.md                   (new agent)
commands/prototype.md                     (new command)
commands/iterate.md                       (new command)
commands/annotate.md                      (new command)
commands/extract-plan.md                  (new command)
```

Dev dependencies (already present, no additions needed):
```bash
# Already in package.json devDependencies — no changes
esbuild ^0.24.0    # hooks bundling
c8 ^11.0.0         # coverage
vitest ^4.1.2      # SDK tests only
```

SDK dependency (already present):
```bash
# In sdk/package.json — bump if needed
@anthropic-ai/claude-agent-sdk ^0.2.86
```

---

## Confidence Assessment

| Area | Confidence | Basis |
|------|------------|-------|
| Zero runtime deps constraint | HIGH | Verified by reading all 17 lib files — not a single external require |
| Regex sufficiency for tag extraction | HIGH | PROJECT.md decision confirmed; `@gsd-` prefix tags in comments are structurally simpler than JSDoc |
| node:test for CJS tests | HIGH | All 47 existing test files confirmed via grep |
| claude-agent-sdk version 0.2.86 | HIGH | Verified via `npm view @anthropic-ai/claude-agent-sdk version` |
| Agent file format | HIGH | 19 existing agents provide exact template |
| esbuild 0.27.4 as current | HIGH | Verified via `npm view esbuild version` |
| Vitest 4.1.2 for SDK tests only | HIGH | Confirmed by sdk/package.json and vitest.config.ts scope |

---

## Sources

- Codebase audit: `/get-shit-done/bin/lib/*.cjs` — all 17 lib files read, zero external requires confirmed
- `/tests/*.test.cjs` — all 47 test files use `node:test`
- `sdk/package.json` — claude-agent-sdk `^0.2.84`, vitest `^3.1.1`
- `npm view @anthropic-ai/claude-agent-sdk version` → `0.2.86` (2026-03-28)
- `npm view esbuild version` → `0.27.4` (2026-03-28)
- `npm view c8 version` → `11.0.0` (2026-03-28)
- [comment-parser npm](https://www.npmjs.com/package/comment-parser) — version 1.4.5, 2.3M weekly downloads (rejected: adds dep)
- [Zero Dependency JavaScript is the Future — Liran Tal](https://lirantal.com/blog/zero-dependency-javascript-is-the-future) — rationale for zero-dep CLI tools
- [CLI Framework Comparison: Commander vs Yargs vs Oclif](https://www.grizzlypeaksoftware.com/library/cli-framework-comparison-commander-vs-yargs-vs-oclif-utxlf9v9) — researched and rejected
- [OpenAI Agents SDK TypeScript](https://openai.github.io/openai-agents-js/) — researched and rejected in favor of claude-agent-sdk already in use
- [Building npm packages compatible with ESM and CJS](https://dev.to/snyk/building-an-npm-package-compatible-with-esm-and-cjs-in-2024-88m) — distribution patterns
