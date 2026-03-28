# Feature Landscape

**Domain:** Code-annotation-driven development CLI framework (GSD Code-First fork)
**Researched:** 2026-03-28

---

## Context

This is a fork of an existing agentic CLI framework (GSD / get-shit-done-cc). The fork adds a "code-first" workflow where developers prototype directly, annotate code with structured `@gsd-tags`, and extract those annotations as planning input. The feature landscape is evaluated against two peer categories:

1. **Spec-driven / annotation-driven development tools** — Kiro, BMAD-METHOD, GitHub Spec Kit, OpenSpec, Intent
2. **Existing TODO/annotation extraction tools** — leasot, fixme, todos, code-notes, tickgit

The fork must add new capabilities while preserving all original GSD commands.

---

## Table Stakes

Features users expect from any annotation-driven CLI planning tool. Missing = product feels incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Structured annotation standard | Without a defined tag vocabulary, annotations are inconsistent and unparseable | Low | ARC tags: `@gsd-context`, `@gsd-decision`, `@gsd-todo`, `@gsd-constraint`, `@gsd-pattern`, `@gsd-ref`, `@gsd-risk`, `@gsd-api` |
| Tag scanner / extractor | Core utility; all peer tools (leasot, fixme, todos, code-notes) provide this as baseline | Low-Med | Regex-based, language-agnostic; output JSON and Markdown |
| Inventory output document | Kiro generates tasks files; BMAD shards plans; users expect extraction to produce a readable artifact | Low | `CODE-INVENTORY.md` showing all tags organized by type and file |
| Prototype command | Entry point for code-first mode; without it users don't know how to start | Low | Spawns `gsd-prototyper` agent with project context |
| Iteration command | Extract → plan → approve → execute loop; this IS the code-first workflow | Med | `iterate` command wrapping the full cycle |
| Human approval gate before execution | All agentic tools (Kiro, OpenSpec, Codex CLI) require this; users won't trust autonomous execution without review | Med | Show generated plan, pause for user confirmation before agent runs executor |
| Annotation command for existing code | Most users have existing code; retroactive annotation is essential for adoption | Med | `annotate` command spawning `gsd-annotator` agent |
| Preserved original GSD commands | Users already using GSD expect all original commands (`discuss`, `plan`, `execute`, etc.) to work unchanged | N/A | Non-negotiable constraint; fork must be additive only |
| Per-phase mode configuration | Different phases warrant different workflows; code-first vs plan-first shouldn't be all-or-nothing | Low | `set-mode` command with modes: `code-first`, `plan-first`, `hybrid` |
| Help documentation for new commands | CLI users immediately type `--help` or `/gsd:help`; undocumented commands cause abandonment | Low | Updated help command listing all new commands with one-line descriptions |

---

## Differentiators

Features that distinguish this tool from generic TODO extractors, Kiro, BMAD, and the original GSD. Not universally expected but high competitive value.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Bidirectional code-plan traceability | Annotations in code link to plans; plans reference back to annotated files. No other CLI tool does this end-to-end in a terminal workflow | Med | `@gsd-ref` tags + cross-linking in `CODE-INVENTORY.md` |
| ARC comment obligation in executor | Modified `gsd-executor` requires agents to add ARC annotations as they implement. Creates a self-documenting codebase automatically | Med | Enforced via agent prompt injection in modified `gsd-executor` |
| Code-based planning mode in planner | `gsd-planner` reads `CODE-INVENTORY.md` as planning input instead of (or alongside) requirements docs. Peer tools separate code and planning artifacts | Med | Modified `gsd-planner` with `code-based` planning mode |
| Deep-plan escape hatch | Code-first isn't always right. `deep-plan` command wraps original `discuss-phase` + `plan-phase` for phases needing upfront reasoning | Low | Preserves methodological flexibility; no peer tool offers explicit mode-switching |
| Language-agnostic extraction | Most annotation tools are language-specific (Java, TypeScript). Regex-based approach works on any text file including config, Markdown, SQL | Low | Core design decision; justified by simplicity over AST parsers |
| Inline risk tagging at point of decision | `@gsd-risk` tags capture risks where they're discovered (in code), not in a separate risk register. Closer to engineering reality | Low | Risks aggregated in `CODE-INVENTORY.md` risk section |
| Pattern capture from implementation | `@gsd-pattern` tags let engineers document emerging patterns as they build, feeding future phase planning with discovered (not assumed) patterns | Low | Addresses the BMAD "decision traceability" gap but from code outward |
| Mixed-mode per phase | Each phase independently configured as `code-first` or `plan-first`. Kiro forces one mode; BMAD is always document-first | Low | `phase_modes` config object in extended config schema |

---

## Anti-Features

Features to deliberately NOT build in v1.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| IDE plugin / editor integration | Scope-creep; requires per-editor maintenance; original GSD explicitly has no editor integrations | Use comments that work in any editor; document convention in README |
| AST-based parsing for tag extraction | High complexity, language-specific, fragile across language versions; regex achieves the same goal for structured comments | Regex extraction in `gsd-tools.cjs`; flag AST upgrade as post-v1 option |
| Visual diff viewer / web UI | Mismatches the CLI-first nature of the tool; Kiro, Intent serve IDE/web audiences; this tool's users are terminal-first | Show diffs inline in terminal via standard `git diff` conventions |
| Automatic commit/push on iteration | Agentic tools that auto-commit without approval lose developer trust fast; Codex CLI shows this is table stakes to be explicit | Always pause at approval gate before any git operations |
| Spec-first enforcement | OpenSpec enforces proposal → apply → archive before any code is written; that's the opposite of code-first's value proposition | Support `deep-plan` as an option, never mandate it |
| Custom tag namespace versioning | Managing `@gsd-v2-context` vs `@gsd-context` versioning is operational overhead that adds no value for v1 | Document that tags are stable; handle migration in a future version if needed |
| Multi-repo / monorepo orchestration | Monorepo tooling is a separate domain problem; adds significant complexity without core workflow benefit | Scope to single repo; document as known limitation |
| Real-time annotation sync / watch mode | Continuous watch adds infrastructure complexity and battery/CPU burden; not expected by CLI tool users | Run extraction on-demand via `extract-plan` command |
| Team collaboration / shared annotation server | Collaboration layer requires auth, sync, and conflict resolution; this is a single-developer CLI tool in v1 | Use git as the collaboration mechanism (annotations live in code, code lives in git) |
| Breaking changes to original GSD commands | Would strand existing users and cause merge conflicts with upstream GSD | All additions are strictly additive; original command files unchanged |

---

## Feature Dependencies

```
@gsd-tags standard (ARC annotation vocabulary)
  └── gsd-tools.cjs tag scanner (reads tags)
        └── extract-plan command (CODE-INVENTORY.md output)
              ├── iterate command (reads inventory as planning input)
              │     ├── code-planner agent (CODE-INVENTORY.md → plan)
              │     │     └── approval gate (human review)
              │     │           └── gsd-executor (modified, ARC obligation)
              │     └── prototype command (spawns gsd-prototyper with ARC obligation)
              └── annotate command (spawns gsd-annotator → runs extract-plan)

set-mode command
  └── phase_modes config (per-phase mode storage)
        ├── code-first mode → prototype / iterate / annotate flow
        ├── plan-first mode → original discuss / plan / execute flow
        └── hybrid mode → both available

deep-plan command
  └── discuss-phase (original, unchanged)
        └── plan-phase (original, unchanged)

Modified gsd-executor (ARC comment obligation)
  └── depends on: @gsd-tags standard

Modified gsd-planner (code-based planning mode)
  └── depends on: CODE-INVENTORY.md (output of extract-plan)
```

**Critical path:** ARC annotation standard must be finalized before tag scanner, which must exist before any command that consumes inventory output.

---

## MVP Recommendation

Prioritize in this order:

1. **ARC annotation standard** — the vocabulary everything else depends on; must be stable before any tooling
2. **Tag scanner in gsd-tools.cjs** — enables everything downstream; low complexity, high leverage
3. **extract-plan command** — first user-visible output; validates the annotation → inventory flow
4. **prototype command + gsd-prototyper agent** — entry point to code-first mode
5. **gsd-annotator agent + annotate command** — unlocks brownfield adoption (existing codebases)
6. **iterate command (full loop)** — the core workflow; requires extract-plan, code-planner, approval gate, executor
7. **Modified gsd-executor and gsd-planner** — enriches existing workflow with ARC obligations
8. **set-mode / deep-plan / per-phase config** — quality-of-life features after core loop is working
9. **Updated installer and help docs** — polish after functionality is stable

**Defer:**
- Any feature requiring AST parsing (flag as v2)
- IDE integrations (explicitly out of scope)
- Multi-repo support (explicitly out of scope)

---

## Sources

- [6 Best Spec-Driven Development Tools for AI Coding in 2026 | Augment Code](https://www.augmentcode.com/tools/best-spec-driven-development-tools)
- [Kiro Specs Documentation](https://kiro.dev/docs/specs/)
- [Introducing Kiro](https://kiro.dev/blog/introducing-kiro/)
- [Comprehensive Guide to Kiro, GitHub Spec Kit, and BMAD-METHOD | Medium](https://medium.com/@visrow/comprehensive-guide-to-spec-driven-development-kiro-github-spec-kit-and-bmad-method-5d28ff61b9b1)
- [BMAD-METHOD GitHub](https://github.com/bmad-code-org/BMAD-METHOD)
- [leasot — parse and output TODOs and FIXMEs](https://github.com/pgilad/leasot)
- [fixme — scan for annotation comments](https://github.com/JohnPostlethwait/fixme)
- [todos — parse TODO comments from code](https://github.com/ianlewis/todos)
- [tickgit — project management in TODO comments](https://www.sitepoint.com/never-forget-a-todo-comment-with-tickgit-your-repos-project-manager/)
- [I Built a Tool That Turns TODO Comments Into Actual Documentation | DEV](https://dev.to/nitish_sharma/i-built-a-tool-that-turns-todo-comments-into-actual-documentation-34d2)
- [OpenSpec — spec-driven framework with approval gates](https://arxiv.org/abs/2602.00180)
- [Codex CLI — plan before change, approve inline](https://developers.openai.com/codex/cli/features)
- [Building Human-In-The-Loop Agentic Workflows | Towards Data Science](https://towardsdatascience.com/building-human-in-the-loop-agentic-workflows/)
- [My LLM codegen workflow | Harper Reed](https://harper.blog/2025/02/16/my-llm-codegen-workflow-atm/)
- [Spec-driven development with AI | GitHub Blog](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
