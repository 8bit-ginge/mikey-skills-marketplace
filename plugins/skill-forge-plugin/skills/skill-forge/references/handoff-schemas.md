# Handoff Schemas

These are the four typed contracts that carry context between phases of the Skill Forge pipeline. Each contract defines the exact fields required and provides a complete worked example using the `meeting-summariser` fictional skill — together they show the full pipeline journey from discovery through validation.

## Table of Contents

- [Contract 1: Skill Spec Document](#contract-1-skill-spec-document) — Phase 1 output, Phase 2 input
- [Contract 2: Prompt Inventory](#contract-2-prompt-inventory) — Phase 2 output, Phase 3 input
- [Contract 3: Optimisation Report](#contract-3-optimisation-report) — Phase 3 output, Phase 4 input
- [Contract 4: Validation Feedback](#contract-4-validation-feedback) — Phase 4 output, routes to Phase 2 or 3
- [Routing Decision Rule](#routing-decision-rule) — Formal IF/THEN rule for Phase 4 routing

---

## Contract 1: Skill Spec Document

The Skill Spec captures everything discovered during Phase 1 and provides Phase 2 with the complete context needed to build the skill without a second interview.

### Structured Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| skill_name | string | yes | Kebab-case skill name (e.g., "meeting-summariser") |
| purpose | string | yes | One-sentence statement of what the skill does and why it exists |
| trigger_phrases | list of strings | yes | Natural language phrases that should activate this skill (minimum 5) |
| not_triggers | list of strings | yes | Phrases that should NOT activate this skill — names specific other skills |
| inputs | list of strings | yes | What the user provides (e.g., "meeting transcript", "agenda") |
| outputs | list of strings | yes | What the skill produces (e.g., "structured summary", "action items") |
| primary_behaviours | list of strings | yes | Core behaviours the skill must exhibit (minimum 3) |
| edge_cases | list of strings | yes | Known edge cases and how to handle them |
| tone_style | string | yes | Voice and style guidance (e.g., "concise, professional, bullet-point-heavy") |
| constraints | list of strings | yes | Hard limits the skill must respect |

### Context Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| design_rationale | string | yes | Why the skill exists — the problem it solves, not just what it does |
| wider_context | string | yes | Ecosystem fit — what other tools/skills it sits alongside, how it complements them |
| unexpected_insights | string | yes | Things the discovery interview surfaced that are not obvious from the purpose alone |
| user_mental_model | string | yes | How the user thinks about the skill — their vocabulary, metaphors, expectations |
| anti_patterns | list of strings | yes | What the skill should explicitly NOT do |
| open_questions | list of strings | yes | Unresolved decisions that Phase 2 should flag if encountered |

### Worked Example: meeting-summariser

```yaml
skill_name: "meeting-summariser"
purpose: "Transforms raw meeting transcripts into structured summaries with action items, decisions, and follow-ups so participants don't need to re-watch recordings."
trigger_phrases:
  - "summarise this meeting"
  - "what happened in the meeting"
  - "extract action items from this transcript"
  - "meeting notes from this call"
  - "turn this transcript into a summary"
  - "what were the key decisions"
not_triggers:
  - "summarise this article"
  - "take notes while I talk"
  - "transcribe this audio"
inputs:
  - "Meeting transcript (text, pasted or uploaded)"
  - "Optional: meeting agenda for context"
  - "Optional: list of attendees and their roles"
outputs:
  - "Structured summary with sections: Overview, Key Decisions, Action Items, Discussion Points, Follow-ups"
  - "Action items formatted as: [Owner] -- [Task] -- [Deadline if mentioned]"
primary_behaviours:
  - "Identify and attribute action items to specific people by name"
  - "Distinguish decisions (resolved) from discussion points (unresolved)"
  - "Preserve the original speaker's intent when summarising — do not editorialize"
  - "Flag items where ownership or deadline is ambiguous"
edge_cases:
  - "Transcript has no clear action items — produce summary without action items section, note absence"
  - "Multiple speakers have the same first name — use last name or role to disambiguate"
  - "Transcript is very short (under 200 words) — produce a brief summary, do not pad"
  - "Transcript contains sensitive information — summarise without redacting (user's responsibility)"
tone_style: "Concise, professional, neutral. Use bullet points over prose. Third person for attributions."
constraints:
  - "Summary must be shorter than the original transcript"
  - "Do not infer action items that were not explicitly stated or clearly implied"
  - "Do not add opinions or recommendations not present in the transcript"
design_rationale: "Meeting recordings exist but nobody re-watches them. The gap is not capture but extraction — turning a 60-minute recording into a 2-minute read with the information people actually need to act on."
wider_context: "Sits alongside calendar and project management tools. The summary output format is designed to paste directly into Slack, Notion, or a project tracker. Does not compete with live transcription tools — it operates on completed transcripts."
unexpected_insights: "Users care most about action item attribution — knowing WHO owns WHAT matters more than a polished prose summary. The biggest frustration with existing tools is action items listed without owners."
user_mental_model: "Users think of this as a 'meeting to action items' converter, not a summariser. The summary sections are secondary to the action item extraction."
anti_patterns:
  - "Do not generate a verbatim transcript reformatted as paragraphs — that is not summarisation"
  - "Do not hallucinate attendee names not present in the transcript"
  - "Do not produce a summary longer than the original input"
open_questions:
  - "Should the skill support multiple output formats (Markdown, plain text, Slack blocks)?"
  - "How should recurring meetings be handled — should it reference previous summaries?"
```

---

## Contract 2: Prompt Inventory

The Prompt Inventory catalogues every prompt element written during Phase 2 with its DAIR technique rationale, giving Phase 3 a complete map for diagnostic review.

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| location | string | yes | File path and line number or section identifier (e.g., "SKILL.md:15" or "SKILL.md :: ## Turn Structure") |
| type | enum | yes | One of: description, instruction, prompt-template, example, error-message |
| content | string | yes | The verbatim text of the prompt element |
| purpose | string | yes | What this element is trying to achieve in the skill's behaviour |
| constraints | string | yes | What limits this element must respect (length, format, tone, etc.) |
| target_behaviour | string | yes | The observable Claude behaviour this element should produce |

### Worked Example: meeting-summariser

The following entries represent the Prompt Inventory produced at the end of Phase 2 for the meeting-summariser skill built from the Skill Spec above.

**Entry 1: Description field**

```yaml
location: "SKILL.md:3 (frontmatter description field)"
type: "description"
content: "Transforms meeting transcripts into structured summaries with action items, decisions, and follow-ups. Use when a user pastes or uploads a meeting transcript and asks for a summary, action items, key decisions, or meeting notes. Does not activate for article summaries, document reviews, or live note-taking requests."
purpose: "Auto-trigger the skill on meeting summary requests; prevent triggering on article summaries or live transcription"
constraints: "Under 1024 characters, no XML angle brackets, third person, must include NOT-triggers"
target_behaviour: "Claude activates meeting-summariser when user pastes a transcript and asks for a summary; does not activate when user asks to summarise an article"
```

**Entry 2: Action item extraction instruction**

```yaml
location: "SKILL.md:28 (## Action Item Extraction)"
type: "instruction"
content: "Extract every action item from the transcript. Format each as: [Owner] -- [Task] -- [Deadline]. Owner must be a person named in the transcript. If no deadline is stated, omit the deadline field entirely. If ownership is ambiguous or no owner is named, use [Unassigned] as the owner. Do not infer owners from context — only attribute to people who are explicitly named in connection with the task."
purpose: "Ensure every action item is attributed to a specific person with a deadline where stated"
constraints: "Must handle cases where no owner is mentioned; must not infer owners not in transcript"
target_behaviour: "Claude produces action items in '[Owner] -- [Task] -- [Deadline]' format, with '[Unassigned]' when owner unclear"
```

**Entry 3: Output format example**

```yaml
location: "SKILL.md:45 (## Output Example)"
type: "example"
content: |
  ## Overview
  The team aligned on Q3 priorities and reviewed the budget proposal.

  ## Key Decisions
  - Approved the revised project timeline (delayed by 2 weeks)
  - Budget proposal to go to finance for final sign-off

  ## Action Items
  - Sarah Chen -- Review Q3 budget proposal -- Friday
  - Marcus -- Update project timeline in Notion -- EOD Tuesday
  - [Unassigned] -- Schedule follow-up with finance team -- TBD

  ## Discussion Points
  - Concerns raised about resource allocation in Q4 (unresolved)

  ## Follow-ups
  - Finance sign-off pending before project can proceed
purpose: "Anchor Claude's output format to the desired structure via few-shot"
constraints: "Must show all sections (Overview, Decisions, Action Items, Discussion, Follow-ups); must be concise enough to fit in SKILL.md"
target_behaviour: "Claude's summaries follow this exact section structure and formatting"
```

---

## Contract 3: Optimisation Report

The Optimisation Report documents every change made during Phase 3, giving Phase 4 a precise change log to validate against test results.

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| entry_location | string | yes | File path and line/section where the change was made (matches Prompt Inventory location format) |
| original | string | yes | Verbatim text before optimisation |
| optimised | string | yes | Verbatim text after optimisation |
| techniques_applied | list of strings | yes | DAIR techniques used in the optimisation (e.g., "few-shot", "chain-of-thought") |
| diagnostic_result | string | yes | What the diagnostic pass flagged that triggered this optimisation |
| changes_summary | string | yes | One-sentence description of what changed and why |

### Worked Example: meeting-summariser

The following entries represent the Optimisation Report produced at the end of Phase 3 for the meeting-summariser skill. The diagnostic pass identified two issues in elements from the Prompt Inventory above.

**Entry 1: Description field — negative trigger specificity**

```yaml
entry_location: "SKILL.md:3 (frontmatter description field)"
original: "Transforms meeting transcripts into structured summaries with action items, decisions, and follow-ups. Use when a user pastes or uploads a meeting transcript and asks for a summary, action items, key decisions, or meeting notes. Does not activate for article summaries, document reviews, or live note-taking requests."
optimised: "Transforms meeting transcripts into structured summaries with action items, decisions, and follow-ups. Use when a user pastes or uploads a meeting transcript and asks for a summary, action items, key decisions, or meeting notes. Does not activate for: article or document summaries (use a summariser skill), live note-taking or transcription requests, or tasks that don't involve an existing transcript."
techniques_applied:
  - "zero-shot clarity improvement"
  - "negative example specification"
diagnostic_result: "Diagnostic flagged: description triggers on 'summarise this document' — the phrase 'document reviews' is too vague and does not name the alternative skill; NOT-triggers lack specificity needed to prevent false activation"
changes_summary: "Replaced vague NOT-trigger categories with explicit named alternatives and clearer scope boundary to prevent false activation on document summary requests"
```

**Entry 2: Action item extraction instruction — unassigned owner handling**

```yaml
entry_location: "SKILL.md:28 (## Action Item Extraction)"
original: "Extract every action item from the transcript. Format each as: [Owner] -- [Task] -- [Deadline]. Owner must be a person named in the transcript. If no deadline is stated, omit the deadline field entirely. If ownership is ambiguous or no owner is named, use [Unassigned] as the owner. Do not infer owners from context — only attribute to people who are explicitly named in connection with the task."
optimised: "Extract every action item from the transcript. Format each as: [Owner] -- [Task] -- [Deadline]. Owner must be a person named in the transcript. If no deadline is stated, omit the deadline field entirely. If ownership is ambiguous or no owner is named, use [Unassigned] as the owner. Do not infer owners from context — only attribute to people who are explicitly named in connection with the task.\n\nWhen multiple attendees share a first name, disambiguate using their last name, role, or team as stated in the transcript or attendee list. Example: 'Sarah Chen (Finance)' or 'Sarah (PM)'."
techniques_applied:
  - "few-shot example addition"
  - "edge case specification"
diagnostic_result: "Diagnostic flagged: no explicit instruction for handling name collisions among attendees — the instruction handles missing owners but not ambiguous owners when two people share a name"
changes_summary: "Added explicit disambiguation instruction and example for same-name attendees, covering the gap between 'unassigned' (no owner) and 'ambiguous' (multiple possible owners)"
```

---

## Contract 4: Validation Feedback

Validation Feedback captures test results and routes issues to the correct phase for correction, packaging enough context that the receiving phase can act without asking the user to re-explain.

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| test_case | string | yes | Description of the test prompt used |
| expected_result | string | yes | What the skill should have produced |
| actual_result | string | yes | What the skill actually produced |
| issue_category | enum | yes | One of: "structural", "prompt-level", "both" |
| issue_description | string | yes | Specific description of what went wrong |
| suggested_route | string | yes | "Phase 2", "Phase 3", or "Phase 2 then Phase 3" (derived from routing decision rule below) |
| re_entry_context | object | yes | Mandatory sub-structure — see re_entry_context section below |

### re_entry_context (required sub-structure)

All four sub-fields are mandatory. No field may be omitted on re-entry. This sub-structure ensures the receiving phase has enough context to act without user reinterpretation.

| Sub-field | Type | Required | Description |
|-----------|------|----------|-------------|
| file_and_line | string | yes | Exact file path and line number where the issue manifests (e.g., "SKILL.md:23") |
| verbatim_output | string | yes | Exact text produced by the failing test case — no paraphrasing |
| expected_output | string | yes | What the test should have produced instead |
| root_cause_hypothesis | string | yes | Best hypothesis for why the output diverged from expected |

### Worked Example: meeting-summariser

This test caught an issue the Phase 3 diagnostic missed — a structural gap in the skill's attendee handling that the optimisation of Entry 2 addressed at the instruction level but did not resolve at the skill architecture level.

```yaml
test_case: "Pasted a transcript where two speakers share the first name 'Sarah'. Asked to extract action items."
expected_result: "Action items attributed using last names or roles to disambiguate (e.g., 'Sarah Chen' or 'Sarah (PM)')"
actual_result: "All action items attributed to 'Sarah' without disambiguation — impossible to tell which Sarah owns which item"
issue_category: "structural"
issue_description: "The skill's SKILL.md ## Action Item Extraction section has an instruction for disambiguation added in Phase 3, but the skill has no mechanism for accepting or using an attendee list with roles — the inputs section lists 'Optional: list of attendees and their roles' but no instruction tells Claude how to cross-reference this input during extraction"
suggested_route: "Phase 2 then Phase 3"
re_entry_context:
  file_and_line: "SKILL.md:28"
  verbatim_output: "Sarah -- Review Q3 budget proposal -- Friday\nSarah -- Update project timeline -- EOD"
  expected_output: "Sarah Chen (Finance) -- Review Q3 budget proposal -- Friday\nSarah Park (PM) -- Update project timeline -- EOD"
  root_cause_hypothesis: "The action item extraction instruction references disambiguation but the skill architecture does not include a step for cross-referencing the optional attendee list input — Claude cannot disambiguate what it has not been instructed to look up"
```

---

## Routing Decision Rule

The following formal decision rule governs how Phase 4 routes issues identified in validation. The rule is applied per Validation Feedback entry.

```
IF issue_category = "structural":
  Route to Phase 2
  After Phase 2 completes, re-run Phase 3 before returning to Phase 4

IF issue_category = "prompt-level":
  Route to Phase 3
  After Phase 3 completes, return to Phase 4

IF issue_category = "both":
  Route to Phase 2 first (structural fix takes precedence)
  After Phase 2 completes, re-run Phase 3 before returning to Phase 4
```

**Rationale:** Structural issues change the skill's architecture (new sections, reorganised flow, missing capabilities). Prompt-level issues are wording or technique problems within an existing structure. When both exist, fixing structure first prevents prompt-level fixes from being invalidated by structural changes.
