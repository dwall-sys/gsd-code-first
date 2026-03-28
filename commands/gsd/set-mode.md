---
name: gsd:set-mode
description: Configure the per-phase workflow mode (code-first, plan-first, hybrid) for gsd-code-first
argument-hint: "<code-first|plan-first|hybrid> [--phase N]"
allowed-tools:
  - Bash
  - Read
---

<objective>
Sets the workflow mode for the current project. Without `--phase`, sets the global default (`default_phase_mode`) applied to all phases. With `--phase N`, overrides the mode for a specific phase only (`phase_modes.N`).

**Arguments:**
- `code-first` — prototype-first workflow (prototype → annotate → extract-plan → iterate)
- `plan-first` — discussion-first workflow (discuss → plan → execute)
- `hybrid` — mix per developer preference
- `--phase N` — scope the mode to phase N only; omit to set the global default

Valid modes: `code-first`, `plan-first`, `hybrid`
</objective>

<context>
$ARGUMENTS
</context>

<process>

1. **Show current mode** so the user knows what they are changing from:
   ```bash
   node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" config-get default_phase_mode
   ```

2. **Parse `$ARGUMENTS`** — extract the mode value (`code-first`, `plan-first`, or `hybrid`) and the optional `--phase N` flag.

3. **Run set-mode** with the parsed arguments:
   - Without `--phase`: `node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" set-mode MODE_VALUE`
   - With `--phase N`: `node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" set-mode MODE_VALUE --phase N`

4. **Show confirmation** with the new mode value and the config key that was written (e.g., `default_phase_mode = code-first` or `phase_modes.3 = plan-first`).

</process>
