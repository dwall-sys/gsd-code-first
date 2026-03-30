# Feature Research

**Domain:** CLI-based autonomous coding workflow — brainstorm-to-PRD, feature map aggregation, architecture-mode prototyping
**Researched:** 2026-03-29
**Confidence:** HIGH (patterns verified across multiple current sources; existing codebase read directly)

---

## Scope Note

This research covers ONLY the new v1.2 features. The v1.0 and v1.1 feature set is treated as the stable baseline and is not re-evaluated here.

The three target features for v1.2 are:

1. **`/gsd:brainstorm`** — interactive conversation → structured PRD(s) with ACs, feature grouping, dependency analysis
2. **Feature Map (`FEATURES.md`)** — auto-aggregated overview from PRDs and `@gsd-tags` in code
3. **Architecture Mode for `/gsd:prototype`** — skeleton-first prototyping for new projects (folder structure, config, interfaces before feature code)

The existing v1.1 pipeline (`/gsd:prototype` → `@gsd-todo` tags → `/gsd:iterate`) is the downstream consumer of all three features. New features must chain into that pipeline without breaking it.

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = the new commands feel broken or incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Brainstorm asks targeted questions before generating a PRD | Every capable AI PRD tool (ChatPRD, Figma AI, Miro AI) uses conversational back-and-forth before generating output. A command that dumps a PRD after one prompt feels unreliable and produces shallow requirements. The Addy Osmani / Bolt "discuss mode" pattern validates this. | MEDIUM | Agent asks one focused question at a time, waits for answer, builds up a picture before drafting. Not a quiz — should feel like a knowledgeable colleague scoping a project. |
| Brainstorm output is a valid `.planning/PRD.md` | Users coming from `/gsd:brainstorm` will immediately run `/gsd:prototype`. If brainstorm output isn't in the exact format prototype expects, the chain breaks and the user has to manually reformat. The two commands must form a pipeline. | LOW | gsd-prototyper already reads `.planning/PRD.md`. Brainstorm must write to the same path with the same AC structure. No new format invention needed. |
| Feature Map is readable as a project status summary | Developers expect a single file that answers "what's in this project and where does it stand?" — similar to how `CODE-INVENTORY.md` answers "what tags exist in this code?" The Feature Map is the high-level counterpart. | MEDIUM | Must aggregate from both PRD ACs and `@gsd-tags` in code. Read-only derived artifact — never edited manually. Regenerated on demand. |
| Architecture Mode produces a runnable (but empty) project skeleton | Tools like the AI-first SDLC scaffold and the Go project skeleton pattern show that architecture-first prototyping is expected to produce real directory structure, config files, and typed interfaces — not documentation stubs. Empty functions and placeholder config are acceptable; non-existent files are not. | MEDIUM | Skeleton includes: project root structure, config files, entry points, typed module interfaces, `@gsd-context` and `@gsd-decision` tags but zero feature implementation. |
| Architecture Mode is invokable via a flag on existing `/gsd:prototype` | Given that `/gsd:prototype` already exists and users are familiar with it, introducing a separate command (`/gsd:scaffold`) adds surface area that duplicates the prototype mental model. Competitors (Bolt, v0) extend existing commands with modes rather than adding parallel commands. | LOW | `--architecture` flag on `/gsd:prototype` triggers skeleton mode. Same command, different execution path inside gsd-prototyper. |
| Feature Map includes completion status per feature | Users tracking multi-feature projects expect to see which ACs are done (implemented in code) vs. still tagged as `@gsd-todo`. Status without completion signal is just a list — not a map. | MEDIUM | Derive from: (a) `@gsd-todo` tags that remain in code = not done; (b) ACs in PRD that have no corresponding `@gsd-todo` = implemented or planned but not annotated. Surface ambiguity explicitly. |

### Differentiators (Competitive Advantage)

Features that set GSD apart from Kiro, spec-kit, Bolt discuss mode, and other tools.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Brainstorm outputs feature-grouped PRD with dependency graph | Bolt/v0 brainstorm produces flat feature lists. Kiro produces linear requirements. GSD's gsd-brainstormer should cluster features into cohesive groups (auth, data model, UI) and surface cross-feature dependencies as text before writing the PRD. This directly informs phase structure in the roadmap. | HIGH | Dependency analysis is the hard part — requires the agent to reason about which features must come before others. Output section: "Feature Groups + Dependencies" before the AC list. The roadmapper agent consumes this output. |
| Feature Map derived from code annotations, not just docs | Kiro, spec-kit, and quantum-loop all track state in external JSON/markdown config files. GSD's differentiator (from v1.0) is that `@gsd-tags` embedded in code ARE the state. FEATURES.md should aggregate from live code annotations — not from a separate state file. This means FEATURES.md is always in sync with the actual code, not a manually maintained artifact. | MEDIUM | `extract-tags` already produces `CODE-INVENTORY.md`. Feature Map aggregator reads CODE-INVENTORY.md plus PRD ACs to produce a higher-level view. No new scanner needed — new aggregation logic only. |
| Architecture Mode annotates structural decisions with `@gsd-decision` tags | Generic scaffolding tools produce folder structure with no explanation for why choices were made. gsd-prototyper in architecture mode embeds `@gsd-decision(phase:0)` tags in config files and module boundaries explaining the architectural choice. This feeds into CODE-INVENTORY.md and makes the "why" of the architecture visible to future agents. | LOW (agent design) | No new tooling — `@gsd-decision` is already in the ARC standard. Agent design choice only. Payoff is high: future `/gsd:iterate` runs have architectural context embedded in the code they're editing. |
| Brainstorm can output multiple scoped PRDs for phased delivery | Large projects in 2026 are built in phases. Brainstorm agents (ChatPRD, Miro, Figma) produce one monolithic PRD. GSD's brainstormer should offer to split output into phase-scoped PRD files (e.g., `PRD-phase-1.md`, `PRD-phase-2.md`) aligned with roadmap phases. Each scoped PRD feeds a separate `/gsd:prototype --phases N` run. | HIGH | Complex: requires the brainstormer to do phase decomposition, not just feature listing. The payoff is substantial — no other tool closes the brainstorm → phased prototype loop end-to-end. May be v1.2+ not v1.2 launch. |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Brainstorm generates code immediately after conversation | Zero-friction — "skip the PRD, just build it" | Without a structured PRD intermediate artifact, there is no checkpoint for human review of requirements before code generation begins. Kiro's linear Requirements → Design → Tasks explicitly prevents this. The human must see and confirm what's being built before the builder starts. Silent scope decisions by the agent compound into major rewrites. | Write PRD, present for confirmation, then user manually runs `/gsd:prototype`. Never auto-chain brainstorm → prototype without a stop. |
| Feature Map is manually editable | Users want to add custom status, notes, flags | Manual edits will be overwritten on the next aggregation run. This creates frustration and loss of human annotations. Every auto-generated artifact in the project (CODE-INVENTORY.md, REVIEW-CODE.md) is derived-only for this reason. | Add a `FEATURES-NOTES.md` sidecar file for human annotations that is never overwritten. FEATURES.md stays derived. |
| Architecture Mode builds a full working app skeleton for any stack | "Just scaffold my Next.js app with auth, DB, and API layer" | Stack-specific scaffold generation is a maintenance burden at the agent layer. There are well-maintained generators for this (create-next-app, create-t3-app). GSD's architecture mode should focus on structural decisions and annotation, not on replacing dedicated scaffolding tools. | Architecture Mode outputs project structure + interface stubs + `@gsd-decision` tags. Delegates actual boilerplate generation to dedicated tools (`npx create-next-app`, etc.) which the agent can invoke. |
| Brainstorm replaces the PRD entirely (code-only output) | Maximum automation | Spec-kit research (Martin Fowler, 2026) explicitly warns that generating code without a human-reviewable spec artifact amplifies AI hallucinations rather than preventing them. The spec (PRD) is the checkpoint. Removing it removes the error-correction surface. | PRD is always the intermediate artifact. It can be minimal (one section, 5 ACs) but it exists and the human sees it. |
| Feature Map shows line-level code coverage per feature | Seems thorough — "which lines implement feature X?" | Requires linking `@gsd-ref` tags in code to PRD ACs with 100% tag discipline. In practice, developers skip tags on refactored code. False "0% coverage" signals are worse than no signal — they erode trust in the entire tool. | Feature Map shows AC-level status (todo tag present / absent) rather than line-level coverage. Simpler, more reliable signal. |

---

## Feature Dependencies

```
[/gsd:brainstorm]
    └──writes──> [.planning/PRD.md]
                     └──required-by──> [/gsd:prototype (existing)]
                     └──required-by──> [Feature Map aggregation]
                                           └──reads-also──> [CODE-INVENTORY.md]
                                                               └──derived-from──> [@gsd-tags in code (existing)]

[/gsd:prototype --architecture]
    └──spawns-variant──> [gsd-prototyper (existing agent, architecture mode)]
                              └──produces──> [project skeleton + @gsd-decision/@gsd-context tags]
                              └──feeds──> [CODE-INVENTORY.md (via existing extract-tags)]
                              └──enables──> [/gsd:prototype (feature mode, next run)]

[Feature Map (/gsd:feature-map OR /gsd:extract-plan --summary)]
    └──requires──> [CODE-INVENTORY.md (from extract-tags)]
    └──requires──> [.planning/PRD.md (for AC list)]
    └──produces──> [.planning/FEATURES.md]

[Existing /gsd:iterate]
    └──unchanged──> reads CODE-INVENTORY.md, no dependency on new features
    └──enhanced-by──> [@gsd-decision tags from architecture mode providing context to gsd-code-planner]
```

### Dependency Notes

- **Brainstorm requires nothing upstream:** It is an entry point — works on a fresh project with no existing planning artifacts. It initializes the PRD that everything else reads.
- **Feature Map requires CODE-INVENTORY.md:** The extract-tags scanner must have run first. Feature Map is a second-pass aggregator, not a primary scanner.
- **Architecture Mode is a flag on existing `/gsd:prototype`:** The existing gsd-prototyper agent handles it. No new agent file needed — architecture mode is a behavioral variant controlled by the `--architecture` flag in the prompt sent to gsd-prototyper.
- **Feature Map and PRD are decoupled from `/gsd:iterate`:** The iterate loop does not need FEATURES.md to function. Feature Map is a summary artifact for human consumption, not an agent input (initially). This keeps the dependency chain shallow.
- **Multi-PRD brainstorm output requires roadmap awareness:** If brainstorm splits into phase-scoped PRDs, the gsd-roadmapper agent must have already produced a ROADMAP.md with phase structure. Brainstorm-with-phase-split is therefore a v1.2+ feature gated on roadmap existence, not a v1.2 launch requirement.

---

## MVP Definition

### Launch With (v1.2)

Minimum viable product for this milestone — what validates the "brainstorm → PRD → prototype → feature map" complete workflow.

- [ ] **`/gsd:brainstorm` command** — conversational agent asks targeted questions, writes `.planning/PRD.md` in the exact format `/gsd:prototype` expects, confirms with user before writing. Single PRD output (no phase splitting at launch).
- [ ] **`gsd-brainstormer` agent** — new agent file. Asks questions one at a time, groups features, identifies cross-feature dependencies, produces PRD with AC list and a brief dependency summary. Writes to `.planning/PRD.md`.
- [ ] **`/gsd:prototype --architecture`** — architecture mode flag on existing command. Spawns gsd-prototyper with architecture mode context: produce skeleton (folder structure, config, entry points, typed interfaces) with `@gsd-decision` and `@gsd-context` tags. Zero feature implementation.
- [ ] **Feature Map command** — new command that reads `CODE-INVENTORY.md` + `.planning/PRD.md` and writes `.planning/FEATURES.md` with feature status (which ACs have `@gsd-todo` tags vs. which are absent from code). Either `/gsd:feature-map` standalone or `--summary` flag on `/gsd:extract-plan`.

### Add After Validation (v1.2+)

- [ ] **Multi-PRD phase splitting in brainstorm** — offer to split output into `PRD-phase-1.md`, `PRD-phase-2.md`, etc., aligned with ROADMAP.md phases. Trigger: users with large projects reporting the monolithic PRD is unwieldy.
- [ ] **Feature Map auto-refresh hook** — regenerate FEATURES.md automatically after each `/gsd:iterate` run. Trigger: users forget to run feature-map manually and see stale status.
- [ ] **`FEATURES-NOTES.md` sidecar** — human-editable companion file that is never overwritten. Trigger: users requesting persistent notes on features.
- [ ] **Brainstorm → feature-map chain** — auto-generate initial FEATURES.md at end of brainstorm run (before any code exists, shows all ACs as pending). Trigger: users wanting to see the full feature list immediately after scoping.

### Future Consideration (v2+)

- [ ] **Spec-as-source mode** — FEATURES.md becomes the source of truth; code is generated from it. Heavy: requires bidirectional sync between spec and code. Martin Fowler's SDD research identifies this as the hardest and most fragile approach.
- [ ] **Feature Map with code coverage links** — link features to specific files/functions. Requires full `@gsd-ref` discipline across the codebase; too fragile to require at v1.2.
- [ ] **Brainstorm from existing codebase** — "I have code, help me write the PRD that describes it." Reverse-engineering intent from code. Brownfield use case. High value but distinct enough to be its own milestone.

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| `/gsd:brainstorm` command + gsd-brainstormer agent | HIGH | MEDIUM | P1 |
| Brainstorm writes valid `.planning/PRD.md` (prototype-compatible) | HIGH | LOW (format already defined) | P1 |
| `/gsd:prototype --architecture` flag | HIGH | LOW (flag + prompt variant in existing agent) | P1 |
| Feature Map command + FEATURES.md output | MEDIUM | MEDIUM | P1 |
| Brainstorm feature grouping + dependency analysis | MEDIUM | MEDIUM (agent reasoning) | P2 |
| Architecture mode `@gsd-decision` annotation discipline | MEDIUM | LOW (agent design choice) | P2 |
| Feature Map completion status (todo tag present/absent) | MEDIUM | LOW (derive from CODE-INVENTORY.md) | P2 |
| Multi-PRD phase splitting | MEDIUM | HIGH | P3 |
| Brainstorm → feature-map auto-chain | LOW | LOW | P3 |

**Priority key:**
- P1: Must have for v1.2 launch
- P2: Should have, include in v1.2 if capacity permits
- P3: Defer to v1.2+ or v2

---

## Competitor Feature Analysis

How competing tools handle the same three domains. None match GSD's code-embedded state model.

| Feature | Bolt discuss mode | Kiro (spec-driven) | spec-kit (GitHub) | GSD v1.2 Approach |
|---------|------------------|--------------------|-------------------|-------------------|
| Brainstorm / scoping | "discuss mode" — conversational without committing code changes; flat feature list output | Requirements → Design → Tasks linear flow; structured but no conversational scoping | Constitution → Specify → Plan → Tasks; constitution defines immutable principles upfront | Conversational questions → feature-grouped PRD with dependency analysis; outputs machine-compatible `.planning/PRD.md` |
| Architecture-first support | None — always feature-first | Design document includes architecture section; not skeleton code | Constitution file defines architecture as text; no skeleton code | `--architecture` flag produces real skeleton code with `@gsd-decision` tags embedded in structure |
| Feature tracking artifact | None (no persistent tracking after scoping) | Spec markdown files per feature; external to code | Many markdown files per branch; spec-anchored but not code-embedded | FEATURES.md derived from live `@gsd-tags` + PRD ACs; always in sync with code state |
| State location | External (brainstorm ends, state lost) | External markdown files | External markdown files | Embedded in code as `@gsd-tags`; CODE-INVENTORY.md is the derived index |
| Brainstorm-to-code chain | Manual: user takes brainstorm output and starts a new build prompt | Manual: user runs Kiro commands sequentially | Manual: user runs spec-kit commands per branch | Explicit: brainstorm writes PRD → user runs `/gsd:prototype` → annotated scaffold produced |
| Human checkpoint | Discuss mode has no explicit confirmation before code generation | Each of Requirements/Design/Tasks are separate steps with human review | Branched workflow implies human review between steps | Explicit confirmation gate: brainstorm shows PRD and asks user to confirm before writing; prototype confirms ACs before building |

**Key GSD differentiation for v1.2:** Kiro and spec-kit both produce specs as external markdown files managed separately from code. GSD's Feature Map is derived from `@gsd-tags` that live IN the code — making FEATURES.md an always-current reflection of code state, not a document that drifts from reality. This is the same architectural bet from v1.0 applied to feature tracking.

---

## Sources

- [Addy Osmani — AI-Driven Prototyping: v0, Bolt, and Lovable Compared](https://addyo.substack.com/p/ai-driven-prototyping-v0-bolt-and) — HIGH confidence (WebFetch verified)
- [Martin Fowler — Understanding Spec-Driven-Development: Kiro, spec-kit, and Tessl](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html) — HIGH confidence (WebFetch verified, Fowler is authoritative)
- [Bolt discuss mode — Bolt Beginners Guide](https://www.nocode.mba/articles/bolt-beginners-guide) — MEDIUM confidence (WebSearch)
- [AI-First SDLC Scaffold — pangon/ai-sdlc-scaffold](https://github.com/pangon/ai-sdlc-scaffold) — MEDIUM confidence (WebSearch, README pattern described)
- [ChatPRD — conversational PRD generation](https://www.chatprd.ai/resources/using-ai-to-write-prd) — MEDIUM confidence (WebSearch)
- [Addy Osmani — LLM coding workflow going into 2026](https://addyosmani.com/blog/ai-coding-workflow/) — HIGH confidence (verified in v1.1 research)
- Existing codebase reads: `/commands/gsd/prototype.md`, `/commands/gsd/iterate.md`, `/commands/gsd/extract-plan.md`, `/agents/gsd-prototyper.md`, `/agents/gsd-code-planner.md`, `/get-shit-done/references/arc-standard.md`, `.planning/PROJECT.md` — HIGH confidence (direct read)
- Existing research: `.planning/research/FEATURES.md` (v1.1) — HIGH confidence (direct read, stable baseline confirmed)

---

*Feature research for: gsd-code-first v1.2 — brainstorm command, feature map, architecture mode*
*Researched: 2026-03-29*
