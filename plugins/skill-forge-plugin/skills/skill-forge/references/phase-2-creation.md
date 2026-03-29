# Phase 2: Creation

This file is loaded when the Skill Forge pipeline enters Phase 2. It provides skill authoring methodology for building a complete skill from a Skill Spec: how to consume the Skill Spec inputs, what element types to produce, how to write and populate the Prompt Inventory concurrently, how to select DAIR techniques for each element type, and how to confirm readiness before advancing to Phase 3.

## Table of Contents

- [Skill Spec Inputs](#skill-spec-inputs)
  - [Structured Fields](#structured-fields)
  - [Context Fields](#context-fields)
- [Element Types](#element-types)
- [Writing Process](#writing-process)
- [DAIR Technique Selection Rubric](#dair-technique-selection-rubric)
  - [Rubric by Element Type](#rubric-by-element-type)
  - [Worked Example: Inventory Entry](#worked-example-inventory-entry)
- [Phase 2 to Phase 3 Gate](#phase-2-to-phase-3-gate)
  - [Claude Self-Assessment](#claude-self-assessment)
  - [User Confirmation](#user-confirmation)

---

## Skill Spec Inputs

Phase 2 opens by consuming the Skill Spec Document handed off from Phase 1. There is no interview, no discovery step, and no follow-up questioning about the user's needs. All 16 fields are treated as pre-verified inputs.

### Structured Fields

The 10 structured fields define what the skill does, what it receives, what it produces, and what constraints it must respect:

- **skill_name** — kebab-case skill name; use as the folder name and frontmatter `name` value
- **purpose** — one-sentence what + why; use to ground the frontmatter description field
- **trigger_phrases** — natural invocation phrases; use these verbatim in the description field trigger list
- **not_triggers** — phrases that should NOT activate this skill; use in the description field NOT-trigger list
- **inputs** — what the user provides; reference in any input-handling instruction blocks
- **outputs** — what the skill produces; anchor output format specification against this list
- **primary_behaviours** — core behaviours the skill must exhibit; map each to an instruction block in SKILL.md
- **edge_cases** — known tricky situations; address each in instruction blocks or as constraint annotations
- **tone_style** — voice and style guidance; apply to any user-facing text and output format instructions
- **constraints** — hard limits; translate directly into constraint fields in Prompt Inventory entries

### Context Fields

The 6 context fields carry the nuanced intent behind the structured fields. They cannot be inferred from the structured fields alone — they come from specific evidence gathered during Phase 1.

**Before writing any skill element, read the full Skill Spec including all 6 context fields:**

- **design_rationale** — why the skill exists; use to frame the purpose and scope of the description field; if the description does not address the design_rationale problem, it is mis-targeted
- **wider_context** — ecosystem fit, what the skill sits alongside; use to define NOT-triggers precisely — adjacent skills and tools the user already has should be named in the NOT-trigger list
- **unexpected_insights** — non-obvious user needs surfaced during discovery; use to anticipate edge cases that are not visible from the structured fields alone
- **user_mental_model** — the user's vocabulary and metaphors for the skill; match their language in trigger phrases and output format labels — not generic skill terminology
- **anti_patterns** — explicit "do not" list from the user; translate directly into constraints fields in the Prompt Inventory and into negative framing in instruction blocks
- **open_questions** — unresolved decisions from Phase 1; if one of these surfaces during writing, flag it explicitly rather than making a silent decision

Reading the structured fields and skipping the context fields produces a technically correct but generic skill. The context fields are what make the skill specific to this user's actual intent.

---

## Element Types

A functional block is a distinct instructional unit — a coherent section of the skill that governs a single behaviour or step. Each functional block gets one Prompt Inventory entry. Typical skills have 4–8 entries. Sub-bullets within an instruction block are NOT separate entries — the block is the entry.

| Element type | What it covers | Typical location |
|---|---|---|
| description | SKILL.md frontmatter description field — the auto-trigger text | `SKILL.md :: frontmatter` |
| instruction | Any directive block telling Claude how to behave | `SKILL.md :: ## [Section name]` |
| prompt-template | A template with `{{variable}}` placeholders for user-facing output | `SKILL.md :: ## [Section name]` |
| example | A worked example (few-shot) demonstrating expected input-output | `SKILL.md :: ## [Section name]` |
| error-message | Text shown when something goes wrong or a gate fails | `SKILL.md :: ## [Section name]` |

The frontmatter description field is always its own entry — it is a distinct prompt element with its own technique profile (see rubric below), its own constraints (1024-char limit, no XML angle brackets, third person), and its own target behaviour (auto-triggering on the right invocation phrases while not triggering on adjacent ones).

Not every element type needs to appear in every skill. A simple skill may have no prompt-template or error-message entries. What matters is that every element you DO write gets its Inventory entry, and that no element type present in the skill is silently skipped.

---

## Writing Process

Build the skill in this order. The order is a suggestion — not a mandate — but it follows a dependency logic: the description field anchors scope, the core instruction blocks deliver the primary behaviours, examples and templates follow once the instruction structure is clear, and error messages fill gaps last.

**Suggested writing order:**
1. description field (frontmatter) — sets scope and trigger profile for everything that follows
2. core instruction blocks — one block per primary behaviour from the Skill Spec
3. output format specification — anchors structure for user-facing output
4. examples — worked few-shot demonstrations of expected input-output
5. error messages or gate conditions — handling failure modes and edge cases

**Concurrent Inventory population rule:**

For each element:
1. Write the element
2. Immediately populate its Prompt Inventory entry:
   - `location` — file path and section identifier (e.g., `"SKILL.md:3 (frontmatter description field)"` or `"SKILL.md :: ## Action Item Extraction"`)
   - `type` — enum value from the Element Types table above
   - `content` — verbatim text just written
   - `purpose` — what this element achieves in the skill's behaviour
   - `constraints` — limits this element must respect (length, format, tone, etc.)
   - `target_behaviour` — the observable Claude behaviour this element should produce
3. Only then begin the next element

This is not a retrospective pass. Do not write the entire SKILL.md and then create the Inventory afterwards. The Inventory's value is in capturing technique decisions at the moment they are made — retrospective annotation produces weaker rationale and collapses the distinction between `purpose` (intent) and `target_behaviour` (observable output).

**`purpose` vs `target_behaviour`:** These are distinct fields. `purpose` answers "What is this element trying to achieve?" — intent-level. `target_behaviour` answers "What would a reviewer SEE Claude do or produce?" — observable output level. A `purpose` that starts with "To ensure..." or "So that..." is correct for its field. The same phrasing in `target_behaviour` is not — `target_behaviour` should describe visible output, not intent.

Refer to references/handoff-schemas.md Contract 2 for the full Prompt Inventory schema and complete worked examples showing all 6 fields populated for the meeting-summariser skill.

---

## DAIR Technique Selection Rubric

For each element you write, select the applicable DAIR techniques before writing and record the selection rationale in the Prompt Inventory entry's `purpose`, `constraints`, and `target_behaviour` fields.

The selection does not need to be exhaustive — apply the techniques that address the element's specific failure modes. An instruction block with no ambiguity does not need chain-of-thought (CoT). An instruction block governing a complex multi-step process does.

### Rubric by Element Type

| Element type | Primary techniques | Selection guidance |
|---|---|---|
| description | zero-shot, negative example specification, role/persona assignment | Zero-shot for the core trigger text. Negative example specification for NOT-trigger phrases — name specific adjacent skills or tasks, not vague categories. Role/persona assignment if the description needs to set expertise context or skill identity. |
| instruction | zero-shot, chain-of-thought (CoT), task decomposition, positive framing, constraint reiteration, uncertainty handling, injection defence | Zero-shot for simple directives. CoT when the instruction governs multi-step reasoning. Task decomposition for sequential processes (step 1, step 2...). Positive framing to rewrite "do not" prohibitions as "do" directives where possible. Constraint reiteration for critical rules that must hold across all runs. Uncertainty handling when the instruction involves factual accuracy or completeness claims. Injection defence if the skill accepts and processes user-provided input. |
| prompt-template | output format specification, output scaffolding | Output format specification to anchor structure, length, and field requirements. Output scaffolding when the template is fill-in-the-blank style (consistent across many instances). |
| example | few-shot examples, output scaffolding | Few-shot examples to anchor consistent output patterns — the example is the technique. Output scaffolding when the example demonstrates a template fill. |
| error-message | zero-shot, positive framing | Zero-shot for simple error or gate-fail messages. Positive framing to guide the user toward the correct action rather than only stating what went wrong. |

### Worked Example: Inventory Entry

See references/handoff-schemas.md Contract 2 for complete worked examples using the meeting-summariser skill. The following is an additional example demonstrating the selection rationale in detail, for an instruction element type:

**Element text (instruction block for a tag-extraction skill):**

```
## Tag Extraction

Scan the full document before generating any tags. Identify candidate terms using these criteria:
1. Appears at least twice, or appears once in a heading
2. Represents a concept, entity, or action — not a stop word or filler
3. Has a clear referent in the document — do not generate abstract descriptors

Output up to 10 tags. If fewer than 3 candidates qualify, output only the candidates that qualify — do not pad to reach 10. Format as a comma-separated list, lowercase, no surrounding punctuation.
```

**Prompt Inventory entry:**

```yaml
location: "SKILL.md :: ## Tag Extraction"
type: "instruction"
content: "[verbatim text above]"
purpose: "Ensure tag extraction is grounded in document content, not generic topic inference; define the output ceiling and formatting to prevent noise"
constraints: "Must handle sparse documents (fewer than 3 qualifying tags) without padding; output must be a comma-separated list with no ambiguity about count or format"
target_behaviour: "Claude scans the full document, produces at most 10 tags in lowercase comma-separated format, outputs fewer tags when fewer qualify, and does not pad with abstract or inferred terms"
```

**Technique selection rationale:**

- **task decomposition** — the three-criteria list (step 1-3) decomposes a judgment call into discrete testable conditions, preventing Claude from applying inconsistent heuristics across runs
- **constraint reiteration** — the output ceiling (10 max), the floor condition (do not pad), and the format (comma-separated, lowercase) are each stated explicitly; they are the kind of rule Claude will drop under pressure if not clearly stated
- **positive framing** applied partially — "do not generate abstract descriptors" is a prohibitive, but it is kept because rephrasing as a positive would be less precise. Constraint reiteration takes precedence when the rule is a hard boundary.

Note how `target_behaviour` describes what a reviewer would see Claude do ("produces at most 10 tags in lowercase comma-separated format, outputs fewer tags when fewer qualify") rather than restating the purpose ("to ground extraction in document content"). These are different fields answering different questions.

---

## Phase 2 to Phase 3 Gate

### Claude Self-Assessment

Before presenting the gate to the user, silently verify all criteria:

1. Skill folder structure exists — at minimum, SKILL.md is present; any reference files mentioned in SKILL.md have been created
2. Prompt Inventory is populated for every prompt element in the skill — no entry has blank `purpose`, `constraints`, or `target_behaviour` fields
3. All element types present in the skill are represented in the Inventory — no element type that was written has been silently omitted from the Inventory
4. Progressive disclosure check passes: SKILL.md body is under 500 lines; any reference files are referenced by name in SKILL.md (not inlined); no reference file references another reference file

If any criterion fails, address the gap before surfacing the gate. The user should never see a failing gate.

---

### User Confirmation

Present the gate to the user:

> "Before we move to diagnostic and polish, I want to confirm three things:
>
> 1. **Skill structure** — [describe what was created: SKILL.md line count, any reference files, folder structure]
> 2. **Prompt Inventory completeness** — [state total entry count, confirm all element types present, flag any entries where constraints or target_behaviour feel thin]
> 3. **Progressive disclosure** — [confirm SKILL.md body is under 500 lines; confirm reference files referenced by name, not inlined]
>
> Does this look right before we run the diagnostic pass?"

Only advance to Phase 3 when the user explicitly confirms. "Looks good" counts. Silence or ambiguity does not — ask again.

On advance: Package the skill files (SKILL.md + any reference files) and the completed Prompt Inventory as the handoff contract to Phase 3. Refer to references/handoff-schemas.md for the full Prompt Inventory schema and field definitions.
