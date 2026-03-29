# Phase 1: Discovery

This file is loaded when the Skill Forge pipeline enters Phase 1. It provides adaptive interview methodology, field-by-field Skill Spec synthesis guidance, a pre-synthesis evidence checklist, and the Phase 1 to Phase 2 confirmation gate.

## Table of Contents

- [Interview Conduct](#interview-conduct)
  - [Turn Rhythm](#turn-rhythm)
  - [Ground Rules](#ground-rules)
  - [Stages](#stages)
  - [Adaptive Direction Framework](#adaptive-direction-framework)
  - [Theme Tracking](#theme-tracking)
- [Pre-Synthesis Checklist](#pre-synthesis-checklist)
- [Synthesis](#synthesis)
  - [Mode Choice](#mode-choice)
  - [Field Walk-Through](#field-walk-through)
  - [Batch Mode Guidance](#batch-mode-guidance)
- [Phase 1 to Phase 2 Gate](#phase-1-to-phase-2-gate)
  - [Claude Self-Assessment](#claude-self-assessment)
  - [User Confirmation](#user-confirmation)

---

## Interview Conduct

### Turn Rhythm

Every turn after the user responds follows this rhythm:

**Reflect** (1-2 sentences) — Mirror what the user said in their own language. This proves you listened and gives them a chance to correct misunderstandings early.

**Connect** (1-2 sentences) — Link their answer to one possibility they may not have considered. Be specific. This is where insight earns trust — but keep it tight.

**Ask** — One question, adapted to what the conversation has revealed. Use the Adaptive Direction Framework to choose where to probe.

**Example turn:**

> **User:** "I keep building one-off Python scripts for data cleaning and then losing them. I want something that remembers my patterns."
>
> **You:** "So the scripts themselves aren't the problem — it's that the knowledge of what to clean and how gets lost between sessions. That's interesting because it means the value isn't in the code, it's in the accumulated decisions. When you say 'remembers my patterns,' are you imagining something that watches you work, or something you explicitly teach?"

---

### Ground Rules

1. **One question per turn.** Occasionally offer 2-4 selectable options when the question suits bounded choices. Open-ended questions are the default.

2. **Start from the problem, not the solution.** Early questions explore what the user does, thinks about, struggles with, and cares about. Skill design details come later.

3. **Explore uncertainty.** When the user doesn't have a clear answer, that's signal. Offer a concrete scenario to react to: *"What if you could..."*

4. **Build on everything.** Track what the user says across turns. Reference earlier answers. Never repeat a question already answered.

5. **Match their register.** Mirror vocabulary, energy, and communication style. Casual user = casual interviewer. Precise user = precise interviewer.

6. **Anchor to invocation.** Regularly check: "When would you actually say this to Claude?" Ground abstract descriptions in concrete trigger moments. A skill that cannot be articulated as a real invocation is not yet well-defined.

---

### Stages

Conduct the interview across 4 stages. Advance based on the transition signal for each stage — not on turn count alone.

| Stage | Name | Duration | Focus | Ready to advance when... |
|-------|------|----------|-------|--------------------------|
| 1 | Opening | 2-3 exchanges | Understand what the skill does and who uses it | "I know the skill's purpose and its user — I don't yet know how it fits their workflow" |
| 2 | Exploration | 3-5 exchanges | Probe workflow integration, edge cases, existing tools, what's missing | "I can describe the user's workflow and the gap this skill fills — I don't yet know the boundaries" |
| 3 | Crystallisation | 3-5 exchanges | Narrow scope, define what's in/out, surface anti-patterns and constraints | "I can state what the skill does and does NOT do — I have enough for most structured fields" |
| 4 | Connection | 2-3 exchanges | Link everything together, fill remaining context field gaps, validate mental model | "I have specific evidence for all 16 Skill Spec fields — ready for pre-synthesis check" |

**Stage 1 — Opening:** If the user has already described their skill idea in their opening message, skip the generic opener — reflect what they said and ask your first probing question immediately.

**Stage 2 — Exploration:** Focus on the user's workflow before and after the skill would fire. What do they do manually that the skill would handle? What does success look like in practice?

**Stage 3 — Crystallisation:** Surface the edges. Ask what the skill should refuse, what it should do differently than something they've tried before, and what would make it annoying or useless.

**Stage 4 — Connection:** Use this stage to fill any remaining gaps in the 6 context fields. These fields require direct evidence from the user — they cannot be inferred. Validate that trigger phrases match how the user would actually invoke the skill by asking them to demonstrate.

---

### Adaptive Direction Framework

After each answer, choose your next question based on the strongest signal present:

| Signal | Where to probe |
|--------|----------------|
| Describes a pain point or friction | What specifically breaks down, when does it happen, and what does it cost them? |
| Mentions a recurring pattern or routine | What does that pattern look like end-to-end? Have they tried to change it? |
| Shows energy or excitement | Follow that thread — what would it look like if it actually worked? |
| Expresses uncertainty | Offer a concrete scenario to react to rather than pushing for an answer |
| Describes a desired end state | Work backwards — what would need to be true for that to exist? |
| Identifies a blocker or constraint | Explore the shape — is it time, knowledge, confidence, resources, or something else? |
| Contradicts something said earlier | Surface the tension gently — it often reveals a deeper insight |
| Describes something they've tried before | What worked, what didn't, and what did they learn? |
| Mentions other people (audience, team, users) | What do those people need, expect, or struggle with? |
| Describes something they admire or want to emulate | What specifically appeals, and what would their version look like? |
| Mentions an existing skill they want to improve | What does the current version do wrong? What specifically needs to change? |
| Describes use in a specific tool or workflow | What comes before invoking this skill? What happens after? |
| Uses ambiguous trigger language ("when I need help with X") | Ask for the last 3 concrete times they would have wanted this |
| Describes what sounds like multiple skills in one | Surface the boundary — which single thing makes this skill succeed or fail? |
| Mentions a skill that already exists (by name) | What does the existing skill NOT do that this one must? |

When multiple signals are present, follow the one the user seems most energised by.

---

### Theme Tracking

Silently track recurring themes across the user's answers. These shape your later questions and become the organising structure of the synthesis.

| Theme | What it sounds like | Maps to Skill Spec fields |
|-------|---------------------|---------------------------|
| **Trigger clarity** | User describes when they'd invoke the skill — how concrete or vague their activation language is | trigger_phrases, not_triggers |
| **Scope ambiguity** | Skill's boundaries are unclear — could be one skill or three, could do too much or too little | primary_behaviours, constraints, edge_cases |
| **Workflow integration** | How the skill fits into the user's existing tools and processes — what comes before and after | wider_context, inputs, outputs |
| **Prior art gaps** | User references existing tools or skills that almost-but-don't-quite solve the problem | design_rationale, not_triggers |
| **Output expectations** | What the user actually wants to receive — format, length, style, level of detail | outputs, tone_style |
| **Identity uncertainty** | User is unsure who the skill is for (themselves, their team, a community) or what role the skill plays | design_rationale, wider_context |

Track which themes appear most frequently — they signal which Skill Spec fields will need the most specific evidence during synthesis. Use the "Maps to Skill Spec fields" column to route theme evidence directly to the corresponding fields.

---

## Pre-Synthesis Checklist

Before beginning synthesis, silently verify you can cite specific, user-quotable evidence for each context field:

1. **design_rationale** — Can you cite a direct quote or specific statement from the user that explains why this skill needs to exist — the problem it solves?
2. **wider_context** — Can you cite the user's own description of what other tools or skills this sits alongside and how it complements them?
3. **unexpected_insights** — Can you point to at least one non-obvious thing the conversation surfaced that is not obvious from the skill's stated purpose?
4. **user_mental_model** — Can you describe how the user thinks about and would invoke this skill, using their vocabulary and metaphors?
5. **anti_patterns** — Can you cite at least one thing the user explicitly said the skill should NOT do?
6. **open_questions** — Have any unresolved decisions surfaced that Phase 2 should be aware of?

If ANY item fails: ask 1-2 targeted follow-up questions to fill the gap. Do not offer the synthesis mode choice until all 6 items pass. This is a hard gate.

*Note: The structured fields (skill_name through constraints) do not need a pre-check — they are populated during synthesis itself. The context fields require pre-existing evidence because they cannot be inferred; they must come from things the user actually said.*

---

## Synthesis

### Mode Choice

Ask the user which synthesis mode they prefer before beginning:

> "I have enough to build your Skill Spec. How would you like to review it?
> (a) Batch — I'll present all 16 fields at once for you to review as a whole
> (b) Field-by-field — I'll propose each field one at a time and you confirm or correct before I move on"

Wait for the user to choose before proceeding.

---

### Field Walk-Through

For each field, identify the relevant evidence from the conversation. Where evidence is direct (the user said it), use it verbatim or near-verbatim. Where evidence must be constructed (e.g., skill_name), derive from the user's language and confirm.

**Structured fields:**

- **skill_name**: Derive from the user's own language. If the user said "something that cleans my data," try `data-cleaner`. Confirm with the user — they may have a better name.
- **purpose**: One sentence capturing what + why. Must be grounded in a user quote, not Claude's summary of the conversation.
- **trigger_phrases**: List at least 5 natural invocations. Draw from moments when the user described when they'd use this. Include variations in formality and specificity.
- **not_triggers**: Name specific existing skills or tasks that are adjacent but distinct. Draw from moments when the user distinguished this from something else.
- **inputs**: What the user provides to the skill. Draw from how the user described their starting material.
- **outputs**: What the skill produces. Draw from the user's description of what they want to receive.
- **primary_behaviours**: At least 3 core behaviours. Draw from what the user described the skill doing.
- **edge_cases**: Known tricky situations. Draw from "what if" moments in the conversation.
- **tone_style**: Voice and style. Draw from how the user described the skill's personality or communication style.
- **constraints**: Hard limits. Draw from things the user said the skill must NOT exceed or violate.

**Context fields:**

- **design_rationale**: Use the evidence gathered in the pre-synthesis checklist. This must be the problem the skill solves, in the user's words — not a restatement of purpose.
- **wider_context**: Use the evidence from the pre-synthesis checklist. Describe ecosystem fit.
- **unexpected_insights**: Use the pre-synthesis checklist evidence. At least one non-obvious insight.
- **user_mental_model**: Use the pre-synthesis checklist evidence. The user's vocabulary and metaphors.
- **anti_patterns**: Use the pre-synthesis checklist evidence. At least one explicit "do not."
- **open_questions**: Any unresolved decisions from the conversation. Phase 2 uses these as flags.

If you cannot point to specific user evidence for any field, do not populate it with an inference — flag it as needing follow-up and ask one targeted question.

---

### Batch Mode Guidance

In Batch mode: after presenting all 16 fields, highlight any field where evidence was thin or inferred rather than directly quoted. Flag these fields with "[thin evidence]" and ask the user to review those specifically. Do not present the entire block as equally solid.

---

## Phase 1 to Phase 2 Gate

### Claude Self-Assessment

Before presenting the gate to the user, silently verify all three criteria:

1. All 10 structured fields have non-placeholder values (every field populated with specific content, not "TBD" or "to be determined")
2. All 6 context fields have specific, user-quotable evidence (not Claude inferences or paraphrases)
3. Trigger phrases reflect how the user would actually invoke this skill (drawn from the user's own language, not generic phrasing)

If any criterion fails, address the gap through targeted questioning before surfacing the gate. The user should never see a failing gate.

---

### User Confirmation

Present the gate to the user:

> "Before we move to building your skill, I want to confirm three things:
>
> 1. **Field completeness** — All 10 structured fields and all 6 context fields are populated. [List any fields you flagged as thin-evidence during synthesis]
> 2. **Evidence quality** — Every context field is grounded in something you actually said, not my interpretation. Please check: design_rationale, wider_context, unexpected_insights, user_mental_model, anti_patterns, open_questions.
> 3. **Trigger accuracy** — The trigger phrases I listed match how you'd actually ask Claude for this. [List the trigger phrases for quick review]
>
> Does this Skill Spec accurately capture what you want? Any fields to adjust?"

Only advance to Phase 2 when the user explicitly confirms. "Looks good" counts. Silence or ambiguity does not — ask again.

On advance: package the completed Skill Spec Document (all 16 fields) as the handoff contract to Phase 2. Refer to references/handoff-schemas.md for the full Skill Spec schema and field definitions.
