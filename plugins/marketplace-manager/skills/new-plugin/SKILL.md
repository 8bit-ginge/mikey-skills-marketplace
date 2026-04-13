---
name: new-plugin
description: "Scaffolds a new plugin container at plugins/ with all required files that pass validation immediately. Creates .claude-plugin/plugin.json, README.md, CHANGELOG.md, documentation/, commands/, and skills/ directories. Validates name is kebab-case and rejects duplicates. Use when creating a new plugin from scratch. Triggers on: new plugin, create plugin, scaffold plugin. Do NOT use to modify existing plugins or create skills."
argument-hint: "[name]"
allowed-tools: Write, Bash
effort: medium
---

# Marketplace: New Plugin

Read: ${CLAUDE_SKILL_DIR}/references/scaffold-templates.md

## How This Command Works

This command scaffolds a complete, valid plugin container using the templates and paths defined in the reference file loaded above. All template content, ecosystem paths, guard check logic, directory creation order, confirmation output format, and silent validation checks are in that reference file.

The scaffolded container is designed to pass `/marketplace-manager:validate` immediately after creation — the templates are derived directly from what validate expects.

## Execution Order

**CRITICAL: Follow this order exactly. Never create files before guard checks pass.**

### Step 1 — Guard Checks

Run all guards from the "Guard Checks" section of scaffold-templates.md. Stop immediately on any failure.

1. Empty argument check — if `$ARGUMENTS` is empty, output error and stop
2. Name validation (SCAF-03) — validate `$ARGUMENTS` against kebab-case regex; output error and stop on failure
3. Existence check (SCAF-04) — verify container does not already exist at plugins directory; output error and stop on failure

No files or directories are created during this step.

### Step 2 — Create Container

Only after all three guards pass, create the plugin container:

1. Create directories using `mkdir -p` in the order from scaffold-templates.md
2. Write `plugin.json` — substitute `NAME` marker with actual `$ARGUMENTS` value
3. Write `README.md` — substitute `NAME` marker with actual `$ARGUMENTS` value
4. Write `CHANGELOG.md` — no substitution needed

### Step 3 — Confirmation and Validation

After all files are created:

1. Run the silent validation checks from the "Silent Validation" section of scaffold-templates.md
2. If all checks pass: output the confirmation file tree and validate summary line (D-04 format)
3. If any check fails: output the full failure report — this indicates a bug in the scaffold template

## Important Rules

- NEVER create files or directories before all guard checks pass
- NEVER hardcode ecosystem paths in this file — all paths come from scaffold-templates.md
- Replace the `NAME` marker in all template content with the actual `$ARGUMENTS` value
- The `name` field in the scaffolded `plugin.json` must match the folder name exactly
- Load scaffold-templates.md via the Read instruction above — never use Glob to find it
