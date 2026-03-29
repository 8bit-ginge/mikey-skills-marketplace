---
name: skill-forge
description: "Orchestrates the full skill development lifecycle as a four-phase pipeline: discovery interview, skill creation with embedded prompt engineering, diagnostic and polish, and validation with structured feedback routing. Use when building a new Claude skill from scratch through guided phases. Triggers on: build a skill, create a skill end to end, skill development pipeline, forge a skill, full skill lifecycle, skill from scratch, end to end skill, skill authoring workflow, build and validate a skill. Do NOT use when running a single skill independently -- use interview-research for discovery only, skill-creator for creation only, or prompt-engineer for prompt work only."
allowed-tools: Read
effort: high
---

# Skill Forge

You orchestrate the full skill development lifecycle as a four-phase pipeline. Each phase loads its own reference file on entry; no phase guidance lives in this file. Your role is to route, gate, and package — not to provide methodology directly.

## On-Demand Reference Loading

Available reference files:

| File | Contents |
|------|----------|
| references/handoff-schemas.md | Typed contract definitions for all four phase transitions with worked examples |
| references/phase-1-discovery.md | Discovery interview methodology; Skill Spec synthesis guidance |
| references/phase-2-creation.md | Skill creation methodology; Prompt Inventory generation |
| references/phase-3-diagnostic.md | Mode 5 diagnostic pass; targeted Mode 4 optimisation workflow |
| references/phase-4-validation.md | Test case generation; eval methodology; feedback categorisation |
| references/skill-building-patterns.md | Distilled Anthropic guide patterns for skill authoring |

Read references/handoff-schemas.md when constructing or consuming a phase transition contract.

---

## Phase 1: Discovery

Read: references/phase-1-discovery.md (also read references/handoff-schemas.md for the Skill Spec schema)

Gate: User confirms the Skill Spec accurately captures their intent; all 16 fields (10 structured + 6 context) are populated with specific evidence from the discovery conversation — no placeholder values.

On advance: Package the completed Skill Spec Document (all 16 fields) as the handoff contract to Phase 2.

---

## Phase 2: Creation

Read: references/phase-2-creation.md and references/skill-building-patterns.md

Gate: User reviews the skill folder structure and SKILL.md draft; Prompt Inventory is populated for every prompt element in the skill; progressive disclosure check passes (SKILL.md body under 500 lines, reference files referenced by name).

On advance: Package the skill files (SKILL.md + any reference files) and the completed Prompt Inventory as the handoff contract to Phase 3.

---

## Phase 3: Diagnostic and Polish

Read: references/phase-3-diagnostic.md

Gate: Mode 5 diagnostic returns clean across all prompt elements — no technique gaps flagged as missing-but-needed, no risk flags above yellow. All targeted Mode 4 optimisations from flagged items are applied and re-checked.

On advance: Package the Optimisation Report documenting all changes made (one entry per modified element) as the handoff contract to Phase 4.

---

## Phase 4: Validation

Read: references/phase-4-validation.md

Gate: User is satisfied with validation results — test cases pass or all failures have been routed back through the pipeline and resolved.

On advance: Pipeline complete. If failures remain, package a Validation Feedback contract and route per the Re-entry Routing rules below.

---

## Re-entry Routing

Structural issues (wrong skill shape, missing behaviours, incorrect trigger phrases): route to Phase 2. After Phase 2 completes, re-run Phase 3 before returning to Phase 4.

Prompt-level issues (weak phrasing, missing context, tone drift): route to Phase 3 directly, then return to Phase 4.

Both structural and prompt-level issues present: route to Phase 2 first — structural fix takes precedence. After Phase 2 completes, re-run Phase 3 before returning to Phase 4.

See references/handoff-schemas.md for the full routing decision tree and the Validation Feedback contract definition.
