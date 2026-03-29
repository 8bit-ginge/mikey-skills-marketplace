# Phase 4: Validation

This file is loaded when the Skill Forge pipeline enters Phase 4. It provides executable guidance for validating the skill: generating test prompts from the Skill Spec via a fixed three-slot formula, running test cases and populating the Validation Feedback Log concurrently, classifying issues using the section-level change test, packaging re-entry context using the mandatory 4-field sub-structure, accumulating the growing log across pipeline loops, and gating completion on zero unresolved entries.

## Table of Contents

- [Inputs](#inputs)
- [Test Case Generation](#test-case-generation)
  - [Three-Slot Formula](#three-slot-formula)
- [Running Test Cases](#running-test-cases)
  - [Concurrent Log Population](#concurrent-log-population)
- [Issue Classification](#issue-classification)
  - [Section-Level Change Test](#section-level-change-test)
- [Routing](#routing)
- [Re-Entry Protocol](#re-entry-protocol)
  - [Growing Log Rule](#growing-log-rule)
  - [Anti-Clean-Slate Rule](#anti-clean-slate-rule)
  - [Full Re-Run Rule](#full-re-run-rule)
- [Phase 4 Completion Gate](#phase-4-completion-gate)
  - [Claude Self-Assessment](#claude-self-assessment)
  - [User Confirmation](#user-confirmation)

---

## Inputs

Phase 4 receives three inputs:

1. **The Optimisation Report from Phase 3** — context on what changed during diagnostic and polish. Used to understand which elements were modified before running tests. Refer to `references/handoff-schemas.md Contract 3` for the full Optimisation Report schema.

2. **The skill files themselves** — SKILL.md and any reference files. These are the test subjects: the test cases are run against the live skill to determine whether it behaves as intended.

3. **On re-entry: the existing Validation Feedback Log** — the accumulated log from prior Phase 4 runs. Do not discard it. Open it and continue from where the prior loop left off.

---

## Test Case Generation

### Three-Slot Formula

Before running any tests, derive test prompts from the Skill Spec using the three-slot formula. Each slot has a defined source — do not generate tests free-form.

**Slot 1 (required) — Core activation test:**
Source: Skill Spec `trigger_phrases`
Select the phrase closest to what a typical user would type naturally — the most representative phrase, not the most specific or most complex.
Purpose: validates that the skill activates and behaves correctly on the primary use case.

**Slot 2 (required) — Edge case test:**
Source: Skill Spec `edge_cases`
Select one item from the `edge_cases` list. Construct a realistic test prompt that exercises that edge condition.
Purpose: validates known edge handling.

**Slot 3 (optional) — Failure probe:**
Source: Skill Spec `edge_cases`, `constraints`, and `primary_behaviours` read together.
Read all three fields. Identify the combination most likely to reveal a gap the current skill implementation may not handle. Craft a prompt targeting that gap.
Purpose: surfaces issues the skill may not handle — tests beyond known edge cases.

For Slot 3 field selection: start with `edge_cases` and `constraints` together. If a gap is apparent from those, target it. Fall back to the most complex behaviour in `primary_behaviours` if no gap is apparent from the first two fields.

---

## Running Test Cases

### Concurrent Log Population

Run test cases in slot order. For each test case:

1. Present the test prompt. Observe the skill's actual output.
2. Immediately populate all 7 Validation Feedback fields for this test case — including the full `re_entry_context` sub-structure if the result diverges from expected.
3. Only then run the next test.

When populating `re_entry_context`, all four sub-fields are mandatory. No sub-field may be omitted:

- `file_and_line` — exact file path and line number where the issue manifests
- `verbatim_output` — exact text produced by the failing test case, no paraphrasing
- `expected_output` — what the test should have produced instead
- `root_cause_hypothesis` — best hypothesis for why the output diverged; state a hypothesis explicitly even when uncertain — a hypothesis is better than a blank field

This is not a retrospective pass. Do not run all test cases first and then write the Validation Feedback Log entries in a batch. Deferred entries lose the immediate context that makes `actual_result` and `root_cause_hypothesis` accurate. A log written after all tests are complete cannot distinguish whether a hypothesis was formed during the test or reconstructed afterwards.

For the full Validation Feedback schema, field definitions, and a complete worked example, refer to `references/handoff-schemas.md Contract 4`.

---

## Issue Classification

### Section-Level Change Test

When a test case produces an actual result that diverges from the expected result, classify the issue using the section-level change test. Apply the test explicitly — do not categorise by prose description of the issue.

**Step 1:** Identify the specific output that diverged from expected.

**Step 2:** Ask: "What change to the skill files would fix this?"

**Step 3:** Apply the classification rules:

- IF fixing the issue requires adding, removing, or reorganising a section of SKILL.md or a reference file → `issue_category = "structural"`
- IF fixing the issue requires changing wording or technique within an existing section (no structural changes to the file) → `issue_category = "prompt-level"`

For the `"both"` category: do not apply additional classification logic here. Use the worked example in `references/handoff-schemas.md Contract 4` to identify when "both" applies. Note that "structural" and "both" produce the same routing outcome — the distinction does not change where the issue is routed.

---

## Routing

After classification, apply the Routing Decision Rule from `references/handoff-schemas.md` to determine which phase receives the issue. Do not reproduce the rule text here — reference it by name and location.

---

## Re-Entry Protocol

### Growing Log Rule

On re-entry to Phase 4, open the existing Validation Feedback Log. Do not create a new one. Prior entries are preserved. Append new entries for any new test cases run. For each prior entry in the log, mark the entry resolved or unresolved based on the re-test result:

- If re-test passes: mark the entry `status: resolved`
- If re-test fails again: mark the entry `status: unresolved` and update the entry's `actual_result` and `re_entry_context` fields with the current test result

The log accumulates across all loops until every entry is marked `status: resolved`.

### Anti-Clean-Slate Rule

Re-entry does not start from a blank slate. Prior Validation Feedback entries are not discarded between loops. Do not create a fresh log on re-entry. Do not omit prior entries from the log. The value of the growing log model is that it tracks the full history of issues, fixes, and re-test outcomes across every pipeline loop — collapsing that history defeats the model.

### Full Re-Run Rule

On Phase 4 re-entry, re-run ALL prior test cases. Run the Slot 1, Slot 2, and Slot 3 tests as originally constructed. Add new tests warranted by the fix (for example, if Phase 2 added a new capability, add a test for it). Prior-round test cases are not dropped.

Rationale: fixes to structural issues or prompt elements can introduce regressions in test cases that previously passed. Running only the newly-fixed issue test misses these regressions.

---

## Phase 4 Completion Gate

### Claude Self-Assessment

Before surfacing the gate to the user, silently verify all criteria:

1. Count all entries in the Validation Feedback Log
2. Verify every entry is marked `status: resolved` — re-testing produced the expected result for every test case
3. Zero unresolved entries is the completion condition
4. If any entry has `status: unresolved`, address the gap (re-route per the Routing Decision Rule) before surfacing the gate

The user should never see a failing gate. If unresolved entries remain, re-route and complete the necessary loop before returning to Phase 4.

---

### User Confirmation

Present the gate to the user:

> "Before we close the pipeline, I want to confirm three things:
>
> 1. **Test cases run** — [state total test cases executed across all loops]
> 2. **Iterations** — [state total loops through the pipeline]
> 3. **Validation Feedback Log** — [confirm all entries are marked resolved; zero unresolved entries remaining]
>
> Does this look right?"

Only complete the pipeline when the user explicitly confirms. "Looks good" counts. Silence or ambiguity does not — ask again.

On advance: Pipeline complete. The Validation Feedback Log (all entries marked `status: resolved`) is the final artifact.
