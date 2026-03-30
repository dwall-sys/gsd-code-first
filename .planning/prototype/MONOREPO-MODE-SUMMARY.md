---
phase: prototype
plan: monorepo-mode
subsystem: monorepo
tags: [monorepo, workspace-detection, manifests, app-scoping]
dependency_graph:
  requires: [arc-scanner, feature-aggregator]
  provides: [workspace-detector, manifest-generator, monorepo-context, monorepo-init]
  affects: [gsd-tools.cjs, extract-tags]
tech_stack:
  added: []
  patterns: [workspace-detection, manifest-generation, two-level-planning]
key_files:
  created:
    - tests/workspace-detector.test.cjs
    - tests/manifest-generator.test.cjs
    - tests/monorepo-context.test.cjs
  modified:
    - get-shit-done/bin/lib/workspace-detector.cjs
    - get-shit-done/bin/lib/manifest-generator.cjs
    - get-shit-done/bin/lib/monorepo-context.cjs
    - get-shit-done/bin/gsd-tools.cjs
    - commands/gsd/monorepo-init.md
decisions:
  - NX project.json discovery scans only first-level subdirectories to avoid deep recursion
  - Two-level glob patterns detect star positions and walk directory levels accordingly
  - .d.ts fallback scan limited to 5 files to bound scan time
metrics:
  duration: 343s
  completed: 2026-03-30
---

# Prototype Plan: Monorepo Mode Implementation Summary

Complete monorepo workspace detection, per-app planning scaffolding, manifest generation, and CLI wiring using zero external dependencies.

## Tasks Completed

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 | Complete workspace-detector.cjs | 083c58d | workspace-detector.cjs |
| 2 | Complete manifest-generator.cjs | bc93897 | manifest-generator.cjs |
| 3 | Complete monorepo-context.cjs | 87bd2af | monorepo-context.cjs |
| 4 | Wire gsd-tools.cjs subcommands | 74d800a | gsd-tools.cjs |
| 5 | Write tests for all 3 CJS modules | e48c47a | 3 test files |
| 6 | Verify monorepo-init.md orchestrator | 1b21572 | monorepo-init.md |

## What Was Implemented

### workspace-detector.cjs
- NX project.json discovery: scans first-level subdirectories for project.json files when no workspaces field exists
- Negation pattern support: !-prefixed globs are skipped
- Two-level glob patterns: packages/*/sub/* walks two directory levels
- validateAppPath confirmed working for both apps and packages

### manifest-generator.cjs
- .d.ts fallback scan: when no barrel/entry file found, scans package root and src/ for .d.ts files (up to 5)
- Deduplicates exports by name across multiple .d.ts files
- safeName encoding (@acme/ui -> acme__ui.md) verified round-trippable with resolveRelevantManifests

### monorepo-context.cjs
- initAppPlanning creates prototype/CODE-INVENTORY.md stub alongside PRD.md and FEATURES.md
- scopeExtractTags null guard confirmed present (ternary on outputFile)
- buildMonorepoContext returns all six MonorepoContext fields

### gsd-tools.cjs
- Added detect-workspace, generate-manifest, generate-manifests, monorepo-init-app subcommands
- Added --app flag to extract-tags with workspace validation and scoped scanning
- Feature-aggregator auto-chain works with app-scoped outputFile

### monorepo-init.md
- Step 3 updated to use concrete `monorepo-init-app` CLI invocation
- All steps reference exact subcommand names registered in gsd-tools.cjs

## Deviations from Plan

None - plan executed exactly as written.

## Test Results

- workspace-detector.test.cjs: 10 tests, 0 failures
- manifest-generator.test.cjs: 8 tests, 0 failures
- monorepo-context.test.cjs: 8 tests, 0 failures
- Total: 26 tests, all passing

## Known Stubs

None - all @gsd-todo stubs addressed.

## Success Criteria Verification

- [x] detectWorkspace identifies NX workspaces using project.json discovery (AC-1)
- [x] extract-tags --app scopes scanning to app directory (AC-4)
- [x] initAppPlanning creates PRD.md, FEATURES.md, and CODE-INVENTORY.md stubs (AC-3)
- [x] generateAllManifests writes one manifest per package (AC-5)
- [x] resolveRelevantManifests matches @acme/ui to acme__ui.md (AC-6)
- [x] buildMonorepoContext returns all six fields (AC-8)
- [x] monorepo-init.md steps match registered subcommand names
- [x] All @gsd-todo items addressed
- [x] All 26 tests pass, zero regressions

## Self-Check: PASSED

All 9 files verified present. All 6 commit hashes verified in git log.
