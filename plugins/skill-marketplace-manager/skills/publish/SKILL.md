---
name: publish
description: "Publishes a skill or plugin to the GitHub marketplace. Runs the full pipeline: validate, version bump, changelog, package, copy to marketplace, regenerate index, git commit and push. The validation gate stops the pipeline if any checks fail — nothing ships broken. Use when you want to publish, release, ship, or push a skill or plugin to the marketplace. Triggers on: publish my skill, release this plugin, ship it, push to marketplace. Do NOT use for validation only or checking status."
argument-hint: "[name]"
allowed-tools: Read, Write, Edit, Bash, Glob
effort: high
---

# Marketplace: Publish

Read: ${CLAUDE_SKILL_DIR}/references/publish-rules.md

## How This Command Works

This command publishes a skill or plugin through a gated 8-stage pipeline. All pipeline logic, ecosystem paths, packaging commands, output format contracts, and failure handling are in the reference file loaded above. SKILL.md is a thin orchestrator — it delegates all execution details to publish-rules.md.

## Input Handling

When `$ARGUMENTS` contains a name:

1. Read the "Type Detection" section from publish-rules.md
2. Determine if the name is a skill or plugin using the Bash existence checks
3. Execute pipeline Stages 1 through 8 in order
4. Print each stage's tick-mark line immediately as it completes (D-01)

When `$ARGUMENTS` is empty:

- Print: `Usage: /marketplace:publish [name]` and stop
- Publish always requires a name — there is no full-ecosystem publish mode

## Pipeline Flow

Execute these stages in sequence. Each stage must succeed before the next begins.

1. **Validation Gate** — run all validation checks from validation-rules.md; hard stop and show full report if any check fails (D-09)
2. **Version Bump Prompt** — read current version, compute three bump options, present the D-04 prompt; wait for user to select 1, 2, or 3
3. **Changelog Summary Prompt** — present "Describe the change in one sentence:"; wait for user response (D-05)
4. **Write Version Bump** — update version field in SKILL.md frontmatter (skills) or plugin.json (plugins)
5. **Write CHANGELOG Entry** — insert new entry at top of CHANGELOG.md in container directory
6. **Package** — ZIP for skills (cd into claude-code-skill/ first to ensure SKILL.md is at root); rsync for plugins
7. **Copy to Marketplace + Update Index** — copy packaged artifact; update marketplace.json for plugins only; regenerate README.md for all artifacts
8. **Git Commit and Push** — stage all changes in marketplace repo, commit with structured message, push to origin/main; capture push exit code and show D-10 recovery if push fails

## Output Format

Follow the output format contracts in publish-rules.md exactly.

- Print each stage tick line immediately after that stage completes — not all at the end
- Use ✅ prefix for each successful stage
- On failure, show all completed ✅ stages then the ❌ failure line
- Follow the exact D-08 failure format for mid-pipeline errors, D-09 for validation failure, D-10 for push failure
- Final success line uses ✔ (heavy check mark) to distinguish the completion summary

## Important Rules

- All ecosystem paths come from publish-rules.md — do NOT hardcode paths in this file
- Load publish-rules.md via the Read instruction above — do NOT use Glob to find it
- Validation gate is non-negotiable — NEVER skip Stage 1 regardless of user request
- Two separate prompts (Stage 2 THEN Stage 3) — never combine into one prompt
- Skills get ZIP packaging (cd into claude-code-skill/ first); plugins get rsync copy — never mix these
- Skills NEVER get marketplace.json entries — only plugins do
- If git push fails, do NOT attempt rollback — show the recovery command per D-10
