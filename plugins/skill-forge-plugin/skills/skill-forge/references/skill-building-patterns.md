# Skill-Building Patterns

This file is loaded alongside phase-2-creation.md when the Skill Forge pipeline enters Phase 2. It provides distilled Anthropic skill-building patterns in a scannable table format for quick reference during skill authoring.

## Table of Contents

- [Progressive Disclosure](#progressive-disclosure)
- [Description Field](#description-field)
  - [Checklist](#checklist)
  - [Patterns](#patterns)
- [Folder Conventions](#folder-conventions)
- [Testing Methodology](#testing-methodology)

---

## Progressive Disclosure

The progressive disclosure model keeps the base context lean. Only Phase-relevant content loads; nothing pre-loads.

| Pattern | Rationale | Common mistake |
|---------|-----------|----------------|
| SKILL.md body under 500 lines | Excessive body loads unnecessary context on every invocation | Inlining phase methodology directly into SKILL.md |
| Reference files loaded on demand via Read tool | Only loads what is needed for the active phase | Pre-loading all reference files at session start |
| All reference files one level deep from SKILL.md | Claude partially reads deeply nested files (head -100), losing content | Having reference files reference other reference files (A to B to C) |
| Load trigger is explicit in SKILL.md | Claude must know what each file contains and when to load it | Glob pattern without a named load trigger — Claude may not know when to read |
| Glob to locate path, then Read to load | Avoids hardcoded paths that break when skill is installed in different locations | Hardcoding absolute or relative paths in load instructions |

---

## Description Field

The description field has the most failure modes of any skill element. Validate with the checklist before treating the field as complete. The 1024-character hard limit and the XML angle-bracket prohibition are platform-enforced constraints — not style preferences.

### Checklist

Before finalising the description field, verify all of the following:

- [ ] Under 1024 characters (count including spaces)
- [ ] No XML angle brackets (< or >) anywhere in the field
- [ ] Written in third person throughout (no "I can help you..." or "You can use this to...")
- [ ] Names at least 3 concrete trigger phrases
- [ ] Includes at least 1 named NOT-trigger (references a specific other skill or task by name)
- [ ] Distinguishes this skill from the most likely confusion skills by name

### Patterns

| Pattern | Rationale | Common mistake |
|---------|-----------|----------------|
| Third-person voice throughout | Hard constraint from official docs; first-person breaks system prompt injection conventions | Writing "I can help you..." or "Use this when you need..." |
| Concrete trigger phrases included in the description text | Skills auto-trigger on matched language — vague descriptions produce false positives and missed activations | Generic phrases like "when you need help" instead of specific invocations |
| Named NOT-triggers that reference specific skills or tasks by name | Prevents false activation; users learn where the skill boundary is | Omitting NOT-triggers or using generic "don't use for unrelated tasks" |
| Character count verification before finalising | Hard limit is 1024 characters including spaces — validation rejects or silently truncates over-length descriptions | Writing until it "feels complete" without checking character count |
| No XML angle brackets even in examples or lists within the description | System prompt injection break — no exceptions, even for illustrative use | Using angle brackets to indicate variable placeholders or schema notation |

---

## Folder Conventions

These conventions are enforced by the platform and by project-level rules. Deviations cause silent failures (wrong casing on SKILL.md, wrong folder name) or distribute unwanted files (evals/ in .skill build).

| Pattern | Rationale | Common mistake |
|---------|-----------|----------------|
| Folder names kebab-case | Enforced by project convention | Using underscores or PascalCase in folder names |
| SKILL.md (case-sensitive) | Exact filename required by the platform — other casings are not recognised | SKILL.MD, skill.md, Skill.md |
| No README.md inside skill folders | Convention violation; all documentation goes in SKILL.md or references/ | Creating README.md for documentation alongside SKILL.md |
| scripts/ alongside references/ | Scripts for executable code only; references/ is for instructional Markdown content | Putting executable scripts inside references/ alongside Markdown files |
| evals/ excluded from .skill build | Test cases are excluded from the ZIP distribution by convention | Accidentally including evals/ in the .skill file distributed to users |

---

## Testing Methodology

Trigger validation is functional testing. A description field that passes the checklist can still fail to activate reliably. Test both activation and non-activation across representative samples.

| Pattern | Rationale | Common mistake |
|---------|-----------|----------------|
| Test on natural invocation phrases, not slash commands | Skills must trigger on conversational language, not direct command invocations | Only testing /skill-name direct invocations and declaring the trigger set validated |
| Test NOT-triggers explicitly | Preventing false activation is as important as ensuring correct triggering | Testing only positive trigger cases; discovering false activations in production |
| Test at least 10 trigger phrases and 10 NOT-trigger phrases | Sample size large enough to surface edge cases and near-misses | Testing 1-2 examples and declaring the trigger set reliable |
| Reload plugins between edits with /reload-plugins | Changes do not take effect without a reload — test results reflect the previous version | Editing SKILL.md and immediately testing without reloading |
| Use --plugin-dir for local development | Tests the actual plugin file without going through an install cycle | Installing the .skill file, iterating, then reinstalling — slow feedback loop |
