# Phase 3: Diagnostic and Polish

This file is loaded when the Skill Forge pipeline enters Phase 3. It provides the diagnostic methodology for auditing all skill elements against the Prompt Inventory and a targeted optimisation protocol for elements that fail the audit. It covers the inputs to Phase 3, the full diagnostic walk-through, severity classification (Green/Yellow/Red), targeted optimisation scoped to Red items only, concurrent Optimisation Report population, a minimal worked example using two Prompt Inventory entries, and the Phase 3 to Phase 4 gate.

## Table of Contents

- [Inputs](#inputs)
- [Diagnostic Walk-Through](#diagnostic-walk-through)
  - [Per-Entry Audit](#per-entry-audit)
  - [Sequencing Rule](#sequencing-rule)
- [Severity Taxonomy](#severity-taxonomy)
- [Targeted Optimisation](#targeted-optimisation)
  - [Anti-Overreach Rule](#anti-overreach-rule)
- [Optimisation Report Population](#optimisation-report-population)
- [Worked Example](#worked-example)
  - [Green Entry](#green-entry)
  - [Red Entry with Optimisation](#red-entry-with-optimisation)
- [Phase 3 to Phase 4 Gate](#phase-3-to-phase-4-gate)
  - [Claude Self-Assessment](#claude-self-assessment)
  - [User Confirmation](#user-confirmation)

---

## Inputs

Phase 3 receives two inputs:

1. **The Prompt Inventory from Phase 2** — the primary audit lens. The Inventory is the specification: each entry records what was written, why it was written, what constraints it must respect, and what observable behaviour it should produce. The diagnostic checks whether the live skill elements match what the Inventory recorded.

2. **The skill files themselves** — SKILL.md and any reference files. These are audited against the Inventory to determine whether live reality matches the recorded specification.

Refer to `references/handoff-schemas.md Contract 2` for the full Prompt Inventory schema and field definitions.

---

## Diagnostic Walk-Through

Walk through every Prompt Inventory entry in order. For each entry, locate the corresponding live element in the skill file and audit it against the entry's own fields. The Prompt Inventory is the spec — the diagnostic checks whether the live element matches what was recorded.

### Per-Entry Audit

Each Prompt Inventory entry is self-describing. Apply three audit checks using the entry's own fields as criteria:

1. **Does the live element achieve `target_behaviour`?** Check the observable output — not general quality or intent. `target_behaviour` describes what a reviewer would SEE Claude do or produce. If the live element would fail that observable test, the check fails.

2. **Does the element still achieve its stated `purpose` and produce the expected `target_behaviour`?** An element that was well-fitted during writing may have drifted as the skill evolved. Check whether the live element genuinely serves the purpose recorded in the Inventory and would produce the target behaviour a reviewer would observe.

3. **Are all `constraints` from the Inventory entry respected in the live element?** Check each constraint individually. A constraint that passes generally but fails on a specific condition is a failure.

**On the `purpose` vs `target_behaviour` distinction:** The audit checks `target_behaviour` — what a reviewer would observe — not `purpose`, which describes intent. A well-written `purpose` starts with "To ensure..." or "So that..."; a correct `target_behaviour` describes visible output ("Claude produces... in format... when..."). The purpose/target_behaviour distinction established during Phase 2 applies directly here: do not collapse the two when evaluating check 1.

### Sequencing Rule

Complete the diagnostic walk-through for ALL Prompt Inventory entries before beginning any optimisation. The full audit must finish first. Do not optimise an element immediately upon finding an issue — record the severity rating and continue to the next entry. Optimisation is a separate pass that runs only after every entry has been audited.

This sequencing constraint is absolute. Beginning any optimisation before all entries are audited violates the diagnostic model. The reason: early optimisation narrows focus to the first Red item and can cause subsequent entries to be audited through the lens of what was just changed, rather than against the original Inventory.

---

## Severity Taxonomy

After completing all three audit checks for an entry, assign one of three severity ratings. Apply the same taxonomy to every element type (description, instruction, prompt-template, example, error-message) — do not create per-type variant criteria.

| Rating | Condition | Triggers Optimisation? | Gate Status |
|--------|-----------|------------------------|-------------|
| **Green** | Element achieves `target_behaviour`; element serves its stated `purpose`; no `constraints` violated | No | Pass |
| **Yellow** | Suboptimal but functional; none of the Red conditions are true; improvements possible but not required | No | Pass |
| **Red** | Any of: (a) element fails to achieve `target_behaviour`, (b) the element does not serve its stated `purpose`, (c) a `constraint` in the Inventory entry is violated | Yes | Block |

**On the Green/Yellow boundary:** Green does not mean optimal — it means the element achieves its target behaviour, serves its stated purpose, and has no constraint violations. An element that could be improved but passes all three audit checks is Green, not Yellow. Yellow is reserved for elements that are functional but where the purpose fit or constraint compliance is noticeably weaker than what Phase 2 intended, without crossing into failure. If in doubt between Green and Yellow, check whether any of the three Red conditions applies — if none do, and the element achieves `target_behaviour`, the rating is Green.

The same three Red conditions apply regardless of element type. An example element, a description element, and an instruction element are all audited against the same three conditions — the element type changes the content, not the standard.

---

## Targeted Optimisation

After the full diagnostic pass is complete, return to each Red-rated entry and apply targeted optimisation. The optimisation addresses the specific failure(s) that caused the Red rating — nothing more.

### Anti-Overreach Rule

Targeted optimisation runs only on Red items. Do not systematically re-optimise elements that received a Green or Yellow rating — regardless of whether improvements seem possible. If you notice improvements to nearby elements while working on a Red item, leave them as-is. The diagnostic pass is complete when all entries have been audited and all Red items have been addressed.

An Optimisation Report entry whose `diagnostic_result` field does not reference a Red flag from the diagnostic pass is evidence of overreach. If you find yourself writing a Report entry for a Green or Yellow item, stop — that entry should not exist.

The rationale for this constraint mirrors the rationale for concurrent Inventory population in Phase 2: Phase 2 already applied prompt engineering principles during writing. Phase 3's value is in catching genuine failures, not in systematically improving what was already well-considered. Treating Green or Yellow items as Red creates churn without diagnostic grounding.

---

## Optimisation Report Population

Populate the Optimisation Report concurrently — one entry after each targeted optimisation. The sequence for each Red item:

1. Apply the targeted optimisation to the Red item in the skill file
2. Immediately populate the Optimisation Report entry with all 6 fields:
   - `entry_location` — must use the same format as the Prompt Inventory `location` field for the same element. Copy the location value from the Inventory entry — do not re-derive it. Mismatched formats break the traceability from Report back to Inventory.
   - `original` — verbatim text before optimisation
   - `optimised` — verbatim text after optimisation
   - `techniques_applied` — DAIR techniques used in the optimisation
   - `diagnostic_result` — what the diagnostic pass flagged that triggered this optimisation (must reference the Red condition — which of the three conditions (a), (b), or (c) was triggered and why)
   - `changes_summary` — one-sentence description of what changed and why
3. Only then move to the next Red item

This is not a retrospective pass. Do not defer Report entries to the end of the optimisation round. The Report's value is in capturing diagnostic reasoning at the moment the fix is applied — deferred entries produce weaker rationale and lose the immediate context that makes `diagnostic_result` accurate. A Report written after all Red items are addressed cannot distinguish which specific observation triggered each fix.

Refer to `references/handoff-schemas.md Contract 3` for the full Optimisation Report schema, field definitions, and two complete worked examples.

---

## Worked Example

The following minimal example illustrates the diagnostic walk-through using two entries from the meeting-summariser Prompt Inventory (handoff-schemas.md Contract 2). One entry goes Green; one goes Red and triggers a targeted optimisation with a Report entry.

### Green Entry

**Inventory entry audited:** Entry 3 — Output format example
(`location: "SKILL.md:45 (## Output Example)"`, `type: "example"`)

Audit check 1 — `target_behaviour`: "Claude's summaries follow this exact section structure and formatting." The live output example shows all five sections (Overview, Key Decisions, Action Items, Discussion Points, Follow-ups) with the correct formatting. **Passes.**

Audit check 2 — `purpose` and `target_behaviour`: The element's purpose is to anchor output format conventions. The live example serves this purpose — it establishes the structural pattern Claude should follow — and the target behaviour (summaries follow this exact structure) would be achieved. **Passes.**

Audit check 3 — `constraints`: "Must show all sections; must be concise enough to fit in SKILL.md." All five sections are present in the example; the example is compact. **Passes.**

**Rating: Green.** The element achieves target behaviour, the technique is appropriate for its purpose, and no constraints are violated. No optimisation needed. No Report entry.

Note that Green means genuinely good — not just "not Red." This example passes all three audit checks cleanly, which is the correct basis for a Green rating, not the absence of Red conditions alone.

### Red Entry with Optimisation

**Inventory entry audited:** Entry 1 — Description field
(`location: "SKILL.md:3 (frontmatter description field)"`, `type: "description"`)

Audit check 1 — `target_behaviour`: "Claude activates meeting-summariser when user pastes a transcript and asks for a summary; does not activate when user asks to summarise an article." Testing against the live description field reveals it triggers on "summarise this document" — the phrase "document reviews" in the NOT-trigger list is too vague and does not name a specific alternative skill. **Fails condition (a): element does not achieve `target_behaviour`.**

Audit check 2 — `purpose` and `target_behaviour`: The element's purpose (trigger activation on transcripts, not other content) is well-served by the description's structure. The techniques used are appropriate for this purpose. **Passes.**

Audit check 3 — `constraints`: Under 1024 characters, no XML angle brackets, third person, must include NOT-triggers. All respected. **Passes.**

**Rating: Red** — fails condition (a): element does not achieve `target_behaviour`.

Targeted optimisation is applied to the description field, addressing the specificity gap in the NOT-trigger list. A Report entry is written immediately after the fix, before auditing the next item. See `references/handoff-schemas.md Contract 3, Entry 1` for the complete Report entry for this optimisation — it demonstrates the concurrent-population flow with all 6 fields populated, including the `diagnostic_result` field referencing the specific Red condition.

---

## Phase 3 to Phase 4 Gate

### Claude Self-Assessment

Before presenting the gate to the user, silently verify all criteria:

1. All Prompt Inventory entries have been audited — none skipped
2. All entries are rated Green or Yellow — no Red entries remain
3. An Optimisation Report entry has been written for every item that received targeted optimisation
4. Every Optimisation Report `entry_location` value matches the Prompt Inventory `location` value for the same element (copied, not re-derived)

If any criterion fails, address the gap before surfacing the gate. The user should never see a failing gate.

---

### User Confirmation

Present the gate to the user:

> "Before we move to validation, I want to confirm three things:
>
> 1. **Diagnostic coverage** — [state total entries audited; confirm none were skipped]
> 2. **Severity breakdown** — [state Green/Yellow/Red counts; confirm no Red entries remain]
> 3. **Optimisation Report** — [state number of elements that received targeted optimisation; summarise the changes made]
>
> Does this look right before we run validation?"

Only advance to Phase 4 when the user explicitly confirms. "Looks good" counts. Silence or ambiguity does not — ask again.

On advance: Package the Optimisation Report documenting all changes made (one entry per modified element) as the handoff contract to Phase 4. Refer to `references/handoff-schemas.md` for the full Optimisation Report schema and field definitions.
