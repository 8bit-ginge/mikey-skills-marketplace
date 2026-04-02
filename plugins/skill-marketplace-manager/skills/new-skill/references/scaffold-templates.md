# Scaffold Templates: New Skill

All template content, ecosystem paths, guard check logic, directory creation order, confirmation output format, and silent validation checks for the `/marketplace:new-skill` command.

## Table of Contents

1. [Ecosystem Paths](#ecosystem-paths)
2. [Guard Checks](#guard-checks)
3. [Skill Container Template](#skill-container-template)
4. [Directory Creation Order](#directory-creation-order)
5. [Confirmation Output](#confirmation-output)
6. [Silent Validation](#silent-validation)

---

## Ecosystem Paths

These absolute paths are the ground truth for locating skills. Do NOT embed these in SKILL.md body — they live here only.

- **Skills directory:** `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/`

Use this path as the base for all guard checks and file creation operations. The variable `SKILLS_DIR` in code examples below refers to this path.

---

## Guard Checks

Both guards MUST pass before any file or directory is created. Run them in this order:

### Guard 1 — Empty Arguments Check

Before any name validation, check if `$ARGUMENTS` is empty:

```
If $ARGUMENTS is empty or whitespace only:
  Output: Error: No name provided. Usage: /marketplace:new-skill [name]
  Stop — create no files or directories
```

### Guard 2 — Name Validation (SCAF-03)

Use `$ARGUMENTS` as the name candidate (trim surrounding whitespace if present).

Validate against kebab-case regex: `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`

This regex accepts:
- All lowercase letters and digits
- Hyphens in the middle (not at start or end)
- Single-character names that are a lowercase letter or digit

```bash
NAME="$ARGUMENTS"
echo "$NAME" | grep -qE '^[a-z0-9]([a-z0-9-]*[a-z0-9])?$' || {
  echo "Error: '$NAME' is not a valid kebab-case name. Use only lowercase letters, numbers, and hyphens. Cannot start or end with a hyphen."
  # Stop — create no files or directories
}
```

On failure output exactly:
`Error: 'NAME' is not a valid kebab-case name. Use only lowercase letters, numbers, and hyphens. Cannot start or end with a hyphen.`

CRITICAL: No files or directories may be created before this check passes.

### Guard 3 — Existence Check (SCAF-04)

After name validation passes, check that the target container directory does not already exist:

```bash
SKILLS_DIR="/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills"
[ -d "$SKILLS_DIR/$NAME" ] && {
  echo "Error: Skill '$NAME' already exists at claude-skills/$NAME/"
  # Stop — create no files or directories
}
```

On failure output exactly:
`Error: Skill 'NAME' already exists at claude-skills/NAME/`

CRITICAL: No files or directories may be created before this check passes.

---

## Skill Container Template

After both guards pass, create these files and directories. Replace `NAME` with the actual `$ARGUMENTS` value throughout. For headings, convert kebab-case to Title Case (e.g., `my-skill` becomes `My Skill`).

### File 1 — `claude-skills/NAME/claude-code-skill/SKILL.md`

This is the starter skill file. Substitute `NAME` for the name field value and the heading. Convert `NAME` to Title Case for the `# Heading` line.

```
---
name: NAME
description: "TODO: Describe what this skill does and when to trigger it. Include what it does, when to use it, and negative triggers. Max 1024 chars."
argument-hint: "[name]"
allowed-tools: Read, Glob, Bash
effort: medium
---

# NAME Title Case

TODO: Add skill instructions.
```

Example: for name `recipe-helper`, the heading becomes `# Recipe Helper`.

### File 2 — `claude-skills/NAME/README.md`

Container-level README with placeholder sections:

```
# NAME

TODO: One-line description of what this skill does.

## Usage

TODO: Describe how to use this skill. Include trigger phrases and example invocations.

## Requirements

- Claude Code 1.0.33+
```

### File 3 — `claude-skills/NAME/CHANGELOG.md`

Keep-a-Changelog format with initial unreleased entry:

```
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]

### Added

- Initial scaffold
```

### Directory 1 — `claude-skills/NAME/claude-desktop-skill/`

Empty directory. Create with `mkdir -p`. No files inside.

---

## Directory Creation Order

After both guard checks pass, create files and directories in this exact order:

1. `mkdir -p claude-skills/NAME/claude-code-skill/`
2. `mkdir -p claude-skills/NAME/claude-desktop-skill/`
3. Write `claude-skills/NAME/claude-code-skill/SKILL.md` — substitute NAME in both the `name:` field and the `# Heading` line
4. Write `claude-skills/NAME/README.md` — substitute NAME in the title heading
5. Write `claude-skills/NAME/CHANGELOG.md` — no substitution needed

All paths are relative to the ecosystem root. Use the absolute path from Ecosystem Paths section.

CRITICAL: Do NOT create any README.md inside `claude-code-skill/`. The validate command checks that no README exists inside the source folder.

---

## Confirmation Output

After all files are created, output this exact format (per D-04):

```
Created: NAME/
  README.md
  CHANGELOG.md
  claude-code-skill/
    SKILL.md
  claude-desktop-skill/

✔ validate: NAME — all checks passed
```

The validate line is the result of the silent validation step (see next section). If silent validation fails, replace the `✔` line with the full failure report.

---

## Silent Validation

After file creation and before outputting the confirmation, run inline validation against the newly created container. These checks mirror Categories 1, 2, and 4 from `skills/validate/references/validation-rules.md`.

Run all 12 checks. If all pass, output the `✔ validate: NAME — all checks passed` line. If any fail, output the full failure report using the same format as `/marketplace:validate`.

### Checks to Run

**Category 1 — Container Structure**

1. Folder exists: `[ -d "$SKILLS_DIR/NAME" ]`
2. `claude-code-skill/` subdirectory exists: `[ -d "$SKILLS_DIR/NAME/claude-code-skill" ]`
3. `SKILL.md` file exists inside `claude-code-skill/`: `[ -f "$SKILLS_DIR/NAME/claude-code-skill/SKILL.md" ]`
4. No README.md inside `claude-code-skill/`: `[ ! -f "$SKILLS_DIR/NAME/claude-code-skill/README.md" ]`
5. Name is kebab-case: already validated by Guard 2 — mark pass
6. `claude-desktop-skill/` subdirectory exists: `[ -d "$SKILLS_DIR/NAME/claude-desktop-skill" ]`

**Category 2 — SKILL.md Frontmatter**

Parse the SKILL.md frontmatter block (content between the first and second `---` delimiters):

7. YAML frontmatter block present: frontmatter delimiters exist
8. `name:` field present and matches folder name: `name: NAME` is in frontmatter block and value equals NAME
9. `description:` field present, under 1024 chars, no XML angle brackets (`<` or `>`)
10. Body line count (lines after closing `---`) is under 500 lines

**Category 4 — Documentation**

11. `README.md` exists at container level: `[ -f "$SKILLS_DIR/NAME/README.md" ]`
12. `CHANGELOG.md` exists at container level: `[ -f "$SKILLS_DIR/NAME/CHANGELOG.md" ]`

### Implementation Note

When implementing these checks in bash, use `PASS=$((PASS + 1))` and `FAIL=$((FAIL + 1))` for counters. Do NOT use `((PASS++))` — it returns exit code 1 when incrementing from 0, which causes false failures in bash scripts.

### Validation Result

- If ALL 12 checks pass: output `✔ validate: NAME — all checks passed`
- If ANY check fails: output the full failure report in the format from `skills/validate/references/validation-rules.md` Output Format Contract (D-03)

**Note:** For a freshly scaffolded container, all 12 checks should always pass. Any failure indicates a bug in the scaffold template — the failure report helps identify which template file needs fixing.

---

*This file is the complete template and logic reference for the `/marketplace:new-skill` command. SKILL.md loads this file explicitly via `Read: ${CLAUDE_SKILL_DIR}/references/scaffold-templates.md`.*
