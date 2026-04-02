---
name: validate
description: "Validates a skill or plugin against Anthropic spec and project conventions, producing a structured pass/fail report with fix guidance for every check. Use when you want to check a skill or plugin before publishing, or to audit the full ecosystem. Triggers on: validate, check my skill, run validation, is my plugin valid, validate all, check everything. Do NOT use for creating new skills or plugins."
argument-hint: "[name]"
allowed-tools: Read, Glob, Bash
effort: medium
---

# Marketplace: Validate

Read: ${CLAUDE_SKILL_DIR}/references/validation-rules.md

## How This Command Works

This command validates skills and plugins against the full rule set defined in the reference file loaded above. All check definitions, pass/fail criteria, fix message templates, and ecosystem paths are in that reference file.

## Input Handling

### Single Artifact Mode

When `$ARGUMENTS` contains a name:

1. Read the "Type Detection" section from validation-rules.md
2. Determine if the name is a skill or plugin using the detection logic
3. Run ALL applicable check categories against the artifact
4. Output the report in the format below

### Full Scan Mode (no argument)

When `$ARGUMENTS` is empty:

1. Read the "Full Scan Mode" section from validation-rules.md
2. Discover all skills and plugins in the ecosystem (skip hidden dirs, .DS_Store)
3. Run ALL applicable check categories for each artifact
4. Output a detailed report for each artifact
5. End with the summary table from validation-rules.md

## Output Format

Follow this format EXACTLY for each artifact validated (per D-03):

```
## validate: <name>

### Container
[one line per check — prefix with checkmark for pass, X for fail]
[if check fails, indent the fix message on the next line]

### SKILL.md Frontmatter
[only for skills — skip this section for plugins]
[one line per check]

### Plugin JSON
[only for plugins — skip this section for skills]
[one line per check]

### Documentation
[one line per check]

---
Result: <N> failures
```

If all checks pass, the result line should read: `Result: all checks passed`

For pass lines, use the format from validation-rules.md (e.g., `description is 1006/1024 chars — 18 chars remaining`).
For fail lines, use the exact fix message template from validation-rules.md.

## Important Rules

- Use the YAML frontmatter parsing method from validation-rules.md (delimiter isolation) — NEVER regex the full file for frontmatter fields
- For description character counting: handle both inline quoted strings AND YAML block scalar format (description: | followed by indented lines)
- Body line count starts AFTER the closing --- delimiter, not from line 1
- All ecosystem paths come from validation-rules.md — do NOT hardcode paths in this file
- Load validation-rules.md via the Read instruction above — do NOT use Glob to find it
