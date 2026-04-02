---
name: new-skill
description: "Scaffolds a new skill container at claude-skills/ with all required files that pass validation immediately. Creates SKILL.md with starter frontmatter, README.md, CHANGELOG.md, and claude-desktop-skill/ directory. Validates name is kebab-case and rejects duplicates. Use when creating a new skill from scratch. Triggers on: new skill, create skill, scaffold skill. Do NOT use to modify existing skills or create plugins."
argument-hint: "[name]"
allowed-tools: Write, Bash
effort: medium
---

# Marketplace: New Skill

Read: ${CLAUDE_SKILL_DIR}/references/scaffold-templates.md

## How This Command Works

This command scaffolds a new skill container using templates and ecosystem paths from the reference file loaded above. All template content, ecosystem paths, guard check logic, and validation checks are in that reference file.

## Execution Order

CRITICAL: Follow this order exactly. Never create files before guards pass.

### Step 1 — Guard Checks

Run both guards to completion before touching the filesystem:

1. **Empty arguments check** — if `$ARGUMENTS` is empty, output the usage error and stop
2. **Name validation (SCAF-03)** — validate `$ARGUMENTS` against the kebab-case regex in the reference file; output the name error and stop if invalid
3. **Existence check (SCAF-04)** — check the target directory does not exist at the skills path; output the duplicate error and stop if it does

No files or directories are created during this step.

### Step 2 — Create Container

Only after both guards pass:

1. Create directories using `mkdir -p`:
   - `claude-skills/NAME/claude-code-skill/`
   - `claude-skills/NAME/claude-desktop-skill/`
2. Write files using templates from the reference file:
   - `claude-skills/NAME/claude-code-skill/SKILL.md` — substitute NAME in both the `name:` field and the `# Heading` line; convert kebab-case NAME to Title Case for the heading
   - `claude-skills/NAME/README.md` — substitute NAME in the title heading
   - `claude-skills/NAME/CHANGELOG.md` — no substitution needed

### Step 3 — Confirmation and Validation

After all files are created:

1. Output the file tree using the confirmation format from the reference file
2. Run silent inline validation using the 12 checks listed in the reference file
3. Output the validation result line

## Important Rules

- NEVER create files or directories before both guard checks pass
- NEVER hardcode ecosystem paths in this file — they come from scaffold-templates.md
- Substitute NAME in all templates before writing — name field in SKILL.md must match the folder name
- name field in the scaffolded SKILL.md must match the folder name exactly
- Load scaffold-templates.md via the Read instruction above — do NOT use Glob to find it
- Do NOT create README.md inside `claude-code-skill/` — validation explicitly checks it does not exist there
