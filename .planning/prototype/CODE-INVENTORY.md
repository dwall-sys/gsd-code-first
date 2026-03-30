[
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/README.md",
    "line": 32,
    "metadata": {},
    "description": "Auth module -- stateless JWT validation, RS256 only",
    "raw": "// @gsd-context Auth module -- stateless JWT validation, RS256 only"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/README.md",
    "line": 33,
    "metadata": {
      "phase": "1"
    },
    "description": "Use jose library for JWT parsing -- zero native deps",
    "raw": "// @gsd-decision(phase:1) Use jose library for JWT parsing -- zero native deps"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/README.md",
    "line": 34,
    "metadata": {
      "phase": "2",
      "priority": "high"
    },
    "description": "Add refresh token rotation",
    "raw": "// @gsd-todo(phase:2, priority:high) Add refresh token rotation"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/README.md",
    "line": 35,
    "metadata": {},
    "description": "Must remain stateless -- no session storage",
    "raw": "// @gsd-constraint Must remain stateless -- no session storage"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/agents/gsd-arc-executor.md",
    "line": 141,
    "metadata": {
      "phase": "2"
    },
    "description": "Add refresh token rotation",
    "raw": "// @gsd-todo(phase:2) Add refresh token rotation"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/agents/gsd-arc-executor.md",
    "line": 150,
    "metadata": {},
    "description": "[description of choice] | rationale: [why this approach was chosen over alternatives]",
    "raw": "// @gsd-decision [description of choice] | rationale: [why this approach was chosen over alternatives]"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/agents/gsd-arc-executor.md",
    "line": 161,
    "metadata": {},
    "description": "Using Map over plain object for O(1) lookup | rationale: data set grows unboundedly",
    "raw": "// @gsd-decision Using Map over plain object for O(1) lookup | rationale: data set grows unboundedly"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/agents/gsd-tester.md",
    "line": 187,
    "metadata": {
      "reason": "external-http-call",
      "severity": "high"
    },
    "description": "sendEmail calls SMTP -- cannot unit test without mocking",
    "raw": "// @gsd-risk(reason:external-http-call, severity:high) sendEmail calls SMTP -- cannot unit test without mocking"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/agents/gsd-tester.md",
    "line": 202,
    "metadata": {
      "reason": "external-http-call",
      "severity": "high"
    },
    "description": "sendEmail() calls SMTP -- cannot be unit tested without mocking",
    "raw": "// @gsd-risk(reason:external-http-call, severity:high) sendEmail() calls SMTP -- cannot be unit tested without mocking"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/agents/gsd-tester.md",
    "line": 203,
    "metadata": {
      "reason": "database-write",
      "severity": "high"
    },
    "description": "deleteUser() issues SQL DELETE -- requires transaction rollback in test setup",
    "raw": "// @gsd-risk(reason:database-write, severity:high) deleteUser() issues SQL DELETE -- requires transaction rollback in test setup"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/agents/gsd-tester.md",
    "line": 204,
    "metadata": {
      "reason": "async-race-condition",
      "severity": "medium"
    },
    "description": "processQueue() may skip items if called concurrently",
    "raw": "// @gsd-risk(reason:async-race-condition, severity:medium) processQueue() may skip items if called concurrently"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/agents/gsd-tester.md",
    "line": 205,
    "metadata": {
      "reason": "browser-api",
      "severity": "low"
    },
    "description": "initAnalytics() calls window.gtag -- not available in Node.js test environment",
    "raw": "// @gsd-risk(reason:browser-api, severity:low) initAnalytics() calls window.gtag -- not available in Node.js test environment"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/commands/gsd/prototype.md",
    "line": 230,
    "metadata": {
      "ref": "AC-1"
    },
    "description": "User can run /gsd:prototype with PRD auto-detection at .planning/PRD.md",
    "raw": "// @gsd-todo(ref:AC-1) User can run /gsd:prototype with PRD auto-detection at .planning/PRD.md"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/commands/gsd/prototype.md",
    "line": 231,
    "metadata": {
      "ref": "AC-3",
      "priority": "high"
    },
    "description": "User is prompted to paste PRD content if no file is found",
    "raw": "// @gsd-todo(ref:AC-3, priority:high) User is prompted to paste PRD content if no file is found"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 1,
    "metadata": {
      "phase": "11"
    },
    "description": "Convention reader utility -- discovers existing project conventions for architecture mode",
    "raw": "// @gsd-context(phase:11) Convention reader utility -- discovers existing project conventions for architecture mode"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 2,
    "metadata": {},
    "description": "Implemented as a standalone CJS module (not inline in the agent) so it can be tested independently",
    "raw": "// @gsd-decision Implemented as a standalone CJS module (not inline in the agent) so it can be tested independently"
  },
  {
    "type": "ref",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 3,
    "metadata": {
      "ref": "ARCH-03"
    },
    "description": "gsd-prototyper reads existing project conventions before generating skeleton",
    "raw": "// @gsd-ref(ref:ARCH-03) gsd-prototyper reads existing project conventions before generating skeleton"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 4,
    "metadata": {},
    "description": "Zero external dependencies -- uses only Node.js built-ins (fs, path)",
    "raw": "// @gsd-constraint Zero external dependencies -- uses only Node.js built-ins (fs, path)"
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 11,
    "metadata": {},
    "description": "readProjectConventions(projectRoot) -- returns ConventionReport object describing discovered patterns",
    "raw": "// @gsd-api readProjectConventions(projectRoot) -- returns ConventionReport object describing discovered patterns"
  },
  {
    "type": "pattern",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 12,
    "metadata": {},
    "description": "Convention reader returns a structured report that the agent prompt can serialize into context",
    "raw": "// @gsd-pattern Convention reader returns a structured report that the agent prompt can serialize into context"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 36,
    "metadata": {
      "ref": "AC-3"
    },
    "description": "Implement full convention discovery: package.json parsing, tsconfig reading, directory pattern detection, linter config extraction",
    "raw": "// @gsd-todo(ref:AC-3) Implement full convention discovery: package.json parsing, tsconfig reading, directory pattern detection, linter config extraction"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 50,
    "metadata": {},
    "description": "Reads package.json for module type, naming conventions, and dependency-based framework detection",
    "raw": "// @gsd-context Reads package.json for module type, naming conventions, and dependency-based framework detection"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 58,
    "metadata": {},
    "description": "Detect test runner from devDependencies keys rather than config files -- faster and covers most cases",
    "raw": "// @gsd-decision Detect test runner from devDependencies keys rather than config files -- faster and covers most cases"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 66,
    "metadata": {},
    "description": "Detect build tool from devDependencies -- covers esbuild, webpack, vite, rollup",
    "raw": "// @gsd-decision Detect build tool from devDependencies -- covers esbuild, webpack, vite, rollup"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 74,
    "metadata": {},
    "description": "Malformed package.json silently ignored -- could produce incorrect convention report",
    "raw": "// @gsd-risk Malformed package.json silently ignored -- could produce incorrect convention report"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 79,
    "metadata": {},
    "description": "Reads TypeScript/JavaScript config for path aliases and module resolution",
    "raw": "// @gsd-context Reads TypeScript/JavaScript config for path aliases and module resolution"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 86,
    "metadata": {},
    "description": "tsconfig.json may have comments (JSONC) -- JSON.parse will fail on comments. Stub ignores this for now.",
    "raw": "// @gsd-risk tsconfig.json may have comments (JSONC) -- JSON.parse will fail on comments. Stub ignores this for now."
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 97,
    "metadata": {},
    "description": "Reads directory names to detect naming convention (kebab-case vs camelCase etc.)",
    "raw": "// @gsd-context Reads directory names to detect naming convention (kebab-case vs camelCase etc.)"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 102,
    "metadata": {},
    "description": "Check for tests/ or __tests__/ directory first, then fall back to checking for colocated .test. files",
    "raw": "// @gsd-decision Check for tests/ or __tests__/ directory first, then fall back to checking for colocated .test. files"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 107,
    "metadata": {},
    "description": "Detect colocated test pattern by scanning for *.test.* files alongside source files",
    "raw": "// @gsd-todo Detect colocated test pattern by scanning for *.test.* files alongside source files"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 110,
    "metadata": {},
    "description": "Checks for linter config files to match code style in generated skeleton",
    "raw": "// @gsd-context Checks for linter config files to match code style in generated skeleton"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 130,
    "metadata": {},
    "description": "Uses readdirSync (not glob) per project zero-dep constraint",
    "raw": "// @gsd-constraint Uses readdirSync (not glob) per project zero-dep constraint"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 159,
    "metadata": {},
    "description": "Simple heuristic: check if majority of directory names match a pattern",
    "raw": "// @gsd-decision Simple heuristic: check if majority of directory names match a pattern"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/convention-reader.cjs",
    "line": 160,
    "metadata": {},
    "description": "Heuristic may misclassify projects with mixed naming -- returns 'unknown' when ambiguous",
    "raw": "// @gsd-risk Heuristic may misclassify projects with mixed naming -- returns 'unknown' when ambiguous"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 10,
    "metadata": {
      "phase": "12"
    },
    "description": "Feature aggregator — reads PRDs and CODE-INVENTORY.md to produce FEATURES.md.",
    "raw": "// @gsd-context(phase:12) Feature aggregator — reads PRDs and CODE-INVENTORY.md to produce FEATURES.md."
  },
  {
    "type": "pattern",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 19,
    "metadata": {},
    "description": "AC lines in PRDs follow the format: AC-N: description text",
    "raw": "// @gsd-pattern AC lines in PRDs follow the format: AC-N: description text"
  },
  {
    "type": "pattern",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 23,
    "metadata": {},
    "description": "Feature group headings in PRDs use ## or ### markdown headers",
    "raw": "// @gsd-pattern Feature group headings in PRDs use ## or ### markdown headers"
  },
  {
    "type": "pattern",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 26,
    "metadata": {},
    "description": "Dependency sections in PRDs use \"## Dependencies\" or \"### Dependencies\"",
    "raw": "// @gsd-pattern Dependency sections in PRDs use \"## Dependencies\" or \"### Dependencies\""
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 32,
    "metadata": {},
    "description": "Parameters: prdContent (string) — raw PRD markdown.",
    "raw": "* @gsd-api Parameters: prdContent (string) — raw PRD markdown."
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 40,
    "metadata": {
      "ref": "AC-1"
    },
    "description": "Implement full PRD parsing — extract ACs, feature groups, and dependency sections from PRD markdown",
    "raw": "// @gsd-todo(ref:AC-1) Implement full PRD parsing — extract ACs, feature groups, and dependency sections from PRD markdown"
  },
  {
    "type": "pattern",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 105,
    "metadata": {},
    "description": "Open @gsd-todo tags with ref:AC-N metadata indicate incomplete ACs.",
    "raw": "// @gsd-pattern Open @gsd-todo tags with ref:AC-N metadata indicate incomplete ACs."
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 112,
    "metadata": {},
    "description": "Parameters: inventoryContent (string) — raw CODE-INVENTORY.md.",
    "raw": "* @gsd-api Parameters: inventoryContent (string) — raw CODE-INVENTORY.md."
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 119,
    "metadata": {
      "ref": "AC-2"
    },
    "description": "Implement CODE-INVENTORY.md parsing to extract open @gsd-todo(ref:AC-N) tags and determine per-AC completion status",
    "raw": "// @gsd-todo(ref:AC-2) Implement CODE-INVENTORY.md parsing to extract open @gsd-todo(ref:AC-N) tags and determine per-AC completion status"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 151,
    "metadata": {},
    "description": "AC completion is derived from tag presence: if a @gsd-todo with ref:AC-N",
    "raw": "// @gsd-decision AC completion is derived from tag presence: if a @gsd-todo with ref:AC-N"
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 158,
    "metadata": {},
    "description": "Parameters: acs (Array), openTodoAcIds (Set<string>).",
    "raw": "* @gsd-api Parameters: acs (Array), openTodoAcIds (Set<string>)."
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 177,
    "metadata": {},
    "description": "Parameters: dependencies (Array<{from, to}>).",
    "raw": "* @gsd-api Parameters: dependencies (Array<{from, to}>)."
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 184,
    "metadata": {
      "ref": "AC-3"
    },
    "description": "Implement dependency visualization in FEATURES.md from PRD dependency sections",
    "raw": "// @gsd-todo(ref:AC-3) Implement dependency visualization in FEATURES.md from PRD dependency sections"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 207,
    "metadata": {},
    "description": "FEATURES.md is a derived read-only artifact. It must never be manually edited.",
    "raw": "// @gsd-constraint FEATURES.md is a derived read-only artifact. It must never be manually edited."
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 213,
    "metadata": {},
    "description": "Parameters: enrichedAcs (Array), dependencies (Array), groups (string[]).",
    "raw": "* @gsd-api Parameters: enrichedAcs (Array), dependencies (Array), groups (string[])."
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 223,
    "metadata": {
      "ref": "AC-5"
    },
    "description": "Generate FEATURES.md as a derived read-only artifact with last-updated timestamp and source-hash header",
    "raw": "// @gsd-todo(ref:AC-5) Generate FEATURES.md as a derived read-only artifact with last-updated timestamp and source-hash header"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 310,
    "metadata": {},
    "description": "PRD discovery uses a simple glob: .planning/PRD.md and .planning/PRD-*.md",
    "raw": "// @gsd-decision PRD discovery uses a simple glob: .planning/PRD.md and .planning/PRD-*.md"
  },
  {
    "type": "pattern",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 336,
    "metadata": {},
    "description": "CLI entry follows arc-scanner.cjs cmdExtractTags pattern:",
    "raw": "// @gsd-pattern CLI entry follows arc-scanner.cjs cmdExtractTags pattern:"
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 345,
    "metadata": {},
    "description": "CLI entry: cmdAggregateFeatures(cwd, opts).",
    "raw": "* @gsd-api CLI entry: cmdAggregateFeatures(cwd, opts)."
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 355,
    "metadata": {
      "ref": "AC-4"
    },
    "description": "Wire aggregate-features into extract-tags auto-chain so FEATURES.md regenerates on every extract-tags run",
    "raw": "// @gsd-todo(ref:AC-4) Wire aggregate-features into extract-tags auto-chain so FEATURES.md regenerates on every extract-tags run"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 365,
    "metadata": {},
    "description": "No PRDs found — FEATURES.md cannot be generated without at least one PRD.",
    "raw": "// @gsd-risk No PRDs found — FEATURES.md cannot be generated without at least one PRD."
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/feature-aggregator.cjs",
    "line": 396,
    "metadata": {},
    "description": "If CODE-INVENTORY.md is missing, all ACs appear \"done\" by default.",
    "raw": "// @gsd-risk If CODE-INVENTORY.md is missing, all ACs appear \"done\" by default."
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 1,
    "metadata": {
      "phase": "11"
    },
    "description": "Skeleton generator -- produces directory tree and file list for architecture mode confirmation gate",
    "raw": "// @gsd-context(phase:11) Skeleton generator -- produces directory tree and file list for architecture mode confirmation gate"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 2,
    "metadata": {},
    "description": "This is a utility that generates the PLAN (tree display), not the files themselves. The agent creates files via Write tool after user confirms the plan.",
    "raw": "// @gsd-decision This is a utility that generates the PLAN (tree display), not the files themselves. The agent creates files via Write tool after user confirms the plan."
  },
  {
    "type": "ref",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 3,
    "metadata": {
      "ref": "ARCH-01"
    },
    "description": "Supports skeleton generation with folder structure, config, and typed interfaces",
    "raw": "// @gsd-ref(ref:ARCH-01) Supports skeleton generation with folder structure, config, and typed interfaces"
  },
  {
    "type": "ref",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 4,
    "metadata": {
      "ref": "ARCH-04"
    },
    "description": "Generates preview for confirmation gate -- no files written until approved",
    "raw": "// @gsd-ref(ref:ARCH-04) Generates preview for confirmation gate -- no files written until approved"
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 10,
    "metadata": {},
    "description": "generateSkeletonPlan(conventions, modules) -- returns SkeletonPlan with tree string and file list",
    "raw": "// @gsd-api generateSkeletonPlan(conventions, modules) -- returns SkeletonPlan with tree string and file list"
  },
  {
    "type": "pattern",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 11,
    "metadata": {},
    "description": "Skeleton plans are data structures, not side-effectful -- file writing is done by the agent after user approval",
    "raw": "// @gsd-pattern Skeleton plans are data structures, not side-effectful -- file writing is done by the agent after user approval"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 38,
    "metadata": {
      "ref": "AC-1"
    },
    "description": "Implement skeleton plan generation that produces folder structure, config files, and typed interfaces based on discovered conventions",
    "raw": "// @gsd-todo(ref:AC-1) Implement skeleton plan generation that produces folder structure, config files, and typed interfaces based on discovered conventions"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 39,
    "metadata": {},
    "description": "Generated skeleton must contain zero feature implementation code -- only structure and interfaces",
    "raw": "// @gsd-constraint Generated skeleton must contain zero feature implementation code -- only structure and interfaces"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 44,
    "metadata": {},
    "description": "Config files are generated first in the plan because they define project-wide conventions that module files depend on",
    "raw": "// @gsd-decision Config files are generated first in the plan because they define project-wide conventions that module files depend on"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 59,
    "metadata": {},
    "description": "Entry point is src/index with extension matching module type (.mjs for ESM, .cjs for CJS, .js as default)",
    "raw": "// @gsd-decision Entry point is src/index with extension matching module type (.mjs for ESM, .cjs for CJS, .js as default)"
  },
  {
    "type": "pattern",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 74,
    "metadata": {},
    "description": "Each module gets exactly three files: barrel (index), types, and a single stub",
    "raw": "// @gsd-pattern Each module gets exactly three files: barrel (index), types, and a single stub"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 75,
    "metadata": {},
    "description": "Three-file module template keeps boundaries consistent and predictable across the codebase",
    "raw": "// @gsd-decision Three-file module template keeps boundaries consistent and predictable across the codebase"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 77,
    "metadata": {},
    "description": "Module naming follows discovered convention (kebab-case, camelCase, etc.)",
    "raw": "// @gsd-context Module naming follows discovered convention (kebab-case, camelCase, etc.)"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 100,
    "metadata": {},
    "description": "Test directory structure matches discovered convention -- colocated or separate",
    "raw": "// @gsd-decision Test directory structure matches discovered convention -- colocated or separate"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 128,
    "metadata": {},
    "description": "Implement naming convention transformations (kebab-case, camelCase, PascalCase, snake_case)",
    "raw": "// @gsd-todo Implement naming convention transformations (kebab-case, camelCase, PascalCase, snake_case)"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 150,
    "metadata": {},
    "description": "Implement tree string builder with proper indentation and box-drawing characters",
    "raw": "// @gsd-todo Implement tree string builder with proper indentation and box-drawing characters"
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/skeleton-generator.cjs",
    "line": 176,
    "metadata": {},
    "description": "Exports: generateSkeletonPlan, applyNamingConvention (for testing)",
    "raw": "// @gsd-api Exports: generateSkeletonPlan, applyNamingConvention (for testing)"
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/bin/lib/test-detector.cjs",
    "line": 9,
    "metadata": {},
    "description": "detectTestFramework(projectRoot: string)",
    "raw": "* @gsd-api detectTestFramework(projectRoot: string)"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 20,
    "metadata": {},
    "description": "JWT validation module — stateless, RS256 only",
    "raw": "// @gsd-context JWT validation module — stateless, RS256 only"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 21,
    "metadata": {
      "phase": "2"
    },
    "description": "Add refresh token rotation",
    "raw": "// @gsd-todo(phase:2) Add refresh token rotation"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 22,
    "metadata": {
      "phase": "2",
      "priority": "high"
    },
    "description": "Add refresh token rotation",
    "raw": "// @gsd-todo(phase:2, priority:high) Add refresh token rotation"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 53,
    "metadata": {},
    "description": "Valid tag — anchored to comment token     (VALID)",
    "raw": "// @gsd-context Valid tag — anchored to comment token     (VALID)"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 60,
    "metadata": {},
    "description": "No external HTTP calls allowed          (VALID — hash comment)",
    "raw": "# @gsd-constraint No external HTTP calls allowed          (VALID — hash comment)"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 62,
    "metadata": {},
    "description": "Partitioned by tenant_id                  (VALID — SQL comment)",
    "raw": "-- @gsd-context Partitioned by tenant_id                  (VALID — SQL comment)"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 85,
    "metadata": {},
    "description": "",
    "raw": "### @gsd-context"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 90,
    "metadata": {
      "phase": "1"
    },
    "description": "Auth middleware — validates JWT on every protected route. Stateless, RS256 only.",
    "raw": "// @gsd-context(phase:1) Auth middleware — validates JWT on every protected route. Stateless, RS256 only."
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 95,
    "metadata": {},
    "description": "",
    "raw": "### @gsd-decision"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 100,
    "metadata": {},
    "description": "Using jose over jsonwebtoken: jose is ESM-compatible and actively maintained. jsonwebtoken has no ESM export.",
    "raw": "// @gsd-decision Using jose over jsonwebtoken: jose is ESM-compatible and actively maintained. jsonwebtoken has no ESM export."
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 105,
    "metadata": {},
    "description": "",
    "raw": "### @gsd-todo"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 110,
    "metadata": {
      "phase": "2",
      "priority": "high"
    },
    "description": "Add refresh token rotation — currently tokens never expire",
    "raw": "// @gsd-todo(phase:2, priority:high) Add refresh token rotation — currently tokens never expire"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 115,
    "metadata": {},
    "description": "",
    "raw": "### @gsd-constraint"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 120,
    "metadata": {},
    "description": "Max response time 200ms — SLA requirement. Do not add synchronous I/O in this path.",
    "raw": "// @gsd-constraint Max response time 200ms — SLA requirement. Do not add synchronous I/O in this path."
  },
  {
    "type": "pattern",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 125,
    "metadata": {},
    "description": "",
    "raw": "### @gsd-pattern"
  },
  {
    "type": "pattern",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 130,
    "metadata": {},
    "description": "Use sync.Once for all singleton initializations in this package — see Init() as the reference implementation",
    "raw": "// @gsd-pattern Use sync.Once for all singleton initializations in this package — see Init() as the reference implementation"
  },
  {
    "type": "ref",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 135,
    "metadata": {},
    "description": "",
    "raw": "### @gsd-ref"
  },
  {
    "type": "ref",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 140,
    "metadata": {
      "ref": "ISSUE-142"
    },
    "description": "Rate limiting logic — see docs/rate-limiting.md for the algorithm specification",
    "raw": "// @gsd-ref(ref:ISSUE-142) Rate limiting logic — see docs/rate-limiting.md for the algorithm specification"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 145,
    "metadata": {},
    "description": "",
    "raw": "### @gsd-risk"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 150,
    "metadata": {
      "ref": "ISSUE-142"
    },
    "description": "Race condition possible if Init() called before DB connection pool is ready",
    "raw": "// @gsd-risk(ref:ISSUE-142) Race condition possible if Init() called before DB connection pool is ready"
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 155,
    "metadata": {},
    "description": "",
    "raw": "### @gsd-api"
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 160,
    "metadata": {},
    "description": "POST /auth/token — body: {email, password} — returns: {token, expiresAt} or 401 on invalid credentials",
    "raw": "// @gsd-api POST /auth/token — body: {email, password} — returns: {token, expiresAt} or 401 on invalid credentials"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 176,
    "metadata": {
      "phase": "2"
    },
    "description": "Single key",
    "raw": "// @gsd-todo(phase:2) Single key"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 177,
    "metadata": {
      "phase": "2",
      "priority": "high"
    },
    "description": "Two keys",
    "raw": "// @gsd-todo(phase:2, priority:high) Two keys"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 178,
    "metadata": {
      "ref": "ISSUE-142",
      "priority": "high"
    },
    "description": "Two keys with external reference",
    "raw": "// @gsd-risk(ref:ISSUE-142, priority:high) Two keys with external reference"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 179,
    "metadata": {
      "phase": "1"
    },
    "description": "Single phase scoping",
    "raw": "// @gsd-context(phase:1) Single phase scoping"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 191,
    "metadata": {
      "phase": "1"
    },
    "description": "Auth middleware — validates JWT on every protected route",
    "raw": "// @gsd-context(phase:1) Auth middleware — validates JWT on every protected route"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 192,
    "metadata": {},
    "description": "Using jose over jsonwebtoken: jose is ESM-compatible, no CommonJS issues",
    "raw": "// @gsd-decision Using jose over jsonwebtoken: jose is ESM-compatible, no CommonJS issues"
  },
  {
    "type": "api",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 197,
    "metadata": {},
    "description": "POST /users — body: {email, password, name} — returns: {id, email, createdAt} or 400/409",
    "raw": "// @gsd-api POST /users — body: {email, password, name} — returns: {id, email, createdAt} or 400/409"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 198,
    "metadata": {},
    "description": "No plaintext passwords stored — bcrypt hash only, cost factor 12",
    "raw": "// @gsd-constraint No plaintext passwords stored — bcrypt hash only, cost factor 12"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 205,
    "metadata": {},
    "description": "No external HTTP calls from this module — must be pure compute",
    "raw": "# @gsd-constraint No external HTTP calls from this module — must be pure compute"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 206,
    "metadata": {
      "phase": "2",
      "priority": "high"
    },
    "description": "Add caching layer for repeated signature verifications",
    "raw": "# @gsd-todo(phase:2, priority:high) Add caching layer for repeated signature verifications"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 215,
    "metadata": {},
    "description": "Using bcrypt not argon2 — bcrypt is available on all target deployment platforms without custom compile",
    "raw": "# @gsd-decision Using bcrypt not argon2 — bcrypt is available on all target deployment platforms without custom compile"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 216,
    "metadata": {},
    "description": "Memory: bcrypt is CPU-bound; under load this blocks the event loop in sync contexts",
    "raw": "# @gsd-risk Memory: bcrypt is CPU-bound; under load this blocks the event loop in sync contexts"
  },
  {
    "type": "pattern",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 224,
    "metadata": {},
    "description": "Use sync.Once for all singleton initializations in this package",
    "raw": "// @gsd-pattern Use sync.Once for all singleton initializations in this package"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 225,
    "metadata": {
      "ref": "ISSUE-142"
    },
    "description": "Race condition possible if Init() called before DB is ready",
    "raw": "// @gsd-risk(ref:ISSUE-142) Race condition possible if Init() called before DB is ready"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 230,
    "metadata": {},
    "description": "Connection pool — shared across all handlers, initialized once at startup",
    "raw": "// @gsd-context Connection pool — shared across all handlers, initialized once at startup"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 231,
    "metadata": {},
    "description": "Max 25 connections — production database limit. Do not increase without DBA approval.",
    "raw": "// @gsd-constraint Max 25 connections — production database limit. Do not increase without DBA approval."
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 238,
    "metadata": {},
    "description": "FFI boundary to the C crypto library — unsafe block intentional",
    "raw": "// @gsd-context FFI boundary to the C crypto library — unsafe block intentional"
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 239,
    "metadata": {},
    "description": "Memory safety: caller must ensure buf lives longer than the returned slice",
    "raw": "// @gsd-risk Memory safety: caller must ensure buf lives longer than the returned slice"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 244,
    "metadata": {},
    "description": "Chose ring over openssl: ring has a smaller attack surface and is pure Rust",
    "raw": "// @gsd-decision Chose ring over openssl: ring has a smaller attack surface and is pure Rust"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 245,
    "metadata": {},
    "description": "FIPS compliance required — ring is FIPS 140-2 validated for production use",
    "raw": "// @gsd-constraint FIPS compliance required — ring is FIPS 140-2 validated for production use"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 252,
    "metadata": {},
    "description": "Partitioned by tenant_id for query isolation — max 50K rows per partition",
    "raw": "-- @gsd-context Partitioned by tenant_id for query isolation — max 50K rows per partition"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 253,
    "metadata": {},
    "description": "No cross-tenant JOINs allowed in this view",
    "raw": "-- @gsd-constraint No cross-tenant JOINs allowed in this view"
  },
  {
    "type": "decision",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 258,
    "metadata": {},
    "description": "Storing UTC timestamps as BIGINT (epoch ms) not TIMESTAMPTZ — avoids timezone conversion bugs in legacy importers",
    "raw": "-- @gsd-decision Storing UTC timestamps as BIGINT (epoch ms) not TIMESTAMPTZ — avoids timezone conversion bugs in legacy importers"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 259,
    "metadata": {
      "phase": "3"
    },
    "description": "Migrate to TIMESTAMPTZ once legacy importers are decommissioned",
    "raw": "-- @gsd-todo(phase:3) Migrate to TIMESTAMPTZ once legacy importers are decommissioned"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 269,
    "metadata": {},
    "description": "Bootstraps the dev environment — must be idempotent",
    "raw": "# @gsd-context Bootstraps the dev environment — must be idempotent"
  },
  {
    "type": "todo",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 270,
    "metadata": {
      "phase": "3"
    },
    "description": "Add --dry-run flag for CI validation without side effects",
    "raw": "# @gsd-todo(phase:3) Add --dry-run flag for CI validation without side effects"
  },
  {
    "type": "constraint",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 275,
    "metadata": {},
    "description": "Requires bash >=4.0 — uses associative arrays. macOS ships bash 3.2; install via Homebrew.",
    "raw": "# @gsd-constraint Requires bash >=4.0 — uses associative arrays. macOS ships bash 3.2; install via Homebrew."
  },
  {
    "type": "risk",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/get-shit-done/references/arc-standard.md",
    "line": 276,
    "metadata": {},
    "description": "If HOME is unset this script silently writes to //.config — add guard before production use",
    "raw": "# @gsd-risk If HOME is unset this script silently writes to //.config — add guard before production use"
  },
  {
    "type": "context",
    "file": "/Users/denniswall/Desktop/GSD-Code-FIrst/gsd-code-first/tests/feature-aggregator.test.cjs",
    "line": 4,
    "metadata": {
      "phase": "12"
    },
    "description": "Unit tests for the feature aggregator module.",
    "raw": "* @gsd-context(phase:12) Unit tests for the feature aggregator module."
  }
]