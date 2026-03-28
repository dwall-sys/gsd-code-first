# Phase 1: Annotation Foundation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-28
**Phase:** 01-annotation-foundation
**Areas discussed:** Tag Syntax, Scanner Architecture, Annotator Scope, Config Schema Placement
**Mode:** Auto (all recommended defaults selected)

---

## Tag Syntax

| Option | Description | Selected |
|--------|-------------|----------|
| Single-line structured format | `@gsd-todo(phase:2) Description` with parenthesized metadata | ✓ |
| Multi-line block format | Block annotations spanning multiple lines | |
| JSDoc-style format | `@gsd-todo {phase:2} Description` with braces | |

**User's choice:** Single-line structured format (auto-selected recommended default)
**Notes:** Anchored to comment tokens to prevent false positives. Metadata in parentheses is optional.

---

## Scanner Architecture

| Option | Description | Selected |
|--------|-------------|----------|
| New lib module (lib/arc-scanner.cjs) | Follows existing module pattern, exposed via gsd-tools subcommand | ✓ |
| Inline in gsd-tools.cjs | Add scanning logic directly to main tool file | |
| Standalone CLI script | Separate Node.js script in bin/ | |

**User's choice:** New lib module in get-shit-done/bin/lib/ (auto-selected recommended default)
**Notes:** Matches existing architecture pattern. Exposed as `extract-tags` subcommand.

---

## Annotator Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Directory-level with file filtering | Annotator receives directory scope, uses glob patterns | ✓ |
| File-by-file | Annotator processes one file at a time | |

**User's choice:** Directory-level with file filtering (auto-selected recommended default)
**Notes:** Matches how gsd-executor works on phase scope.

---

## Config Schema Placement

| Option | Description | Selected |
|--------|-------------|----------|
| Extend existing config.json | Add `arc` and `phase_modes` top-level keys | ✓ |
| Separate arc-config.json | New config file for ARC-specific settings | |

**User's choice:** Extend existing config.json schema (auto-selected recommended default)
**Notes:** Consistent with existing GSD config pattern. Extend lib/config.cjs validation.

---

## Claude's Discretion

- Regex pattern specifics for edge case handling
- CODE-INVENTORY.md exact formatting
- arc-standard.md language examples beyond core 4
- Test fixture design

## Deferred Ideas

None
