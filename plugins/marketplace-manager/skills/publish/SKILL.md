---
name: publish
description: "Publishes a skill or plugin to the GitHub marketplace. Runs the full pipeline: validate, version bump, changelog, package, copy to marketplace, regenerate index, git commit and push. The validation gate stops the pipeline if any checks fail — nothing ships broken. Use when you want to publish, release, ship, or push a skill or plugin to the marketplace. Triggers on: publish my skill, release this plugin, ship it, push to marketplace. Do NOT use for validation only or checking status."
argument-hint: "[name] [--dry-run]"
allowed-tools: Read, Write, Edit, Bash, Glob
effort: high
---

# Marketplace: Publish

Read: ${CLAUDE_SKILL_DIR}/references/publish-rules.md

## How This Command Works

This command publishes a skill or plugin through a gated 9-stage pipeline. All pipeline logic, ecosystem paths, packaging commands, output format contracts, and failure handling are in the reference file loaded above. SKILL.md is a thin orchestrator — it delegates all execution details to publish-rules.md.

## Input Handling

Parse `$ARGUMENTS` as a whitespace-split token list. Classify each token:

- **First non-flag token** → the artifact name
- **Token equal to `--dry-run`** → set MODE to `dry-run` (otherwise MODE is `execute`) — invokes the full pipeline in simulation mode per D-12
- **Any other `--<word>` token** → print one line: `[WARN] Unknown flag: <flag> — ignoring` and continue (do NOT hard-stop; future flags must not break existing invocations)

If no artifact name is present in `$ARGUMENTS` (including the case where only `--dry-run` was supplied):

- Print: `Usage: /marketplace-manager:publish [name] [--dry-run]`
- Stop — publish always requires a name, there is no full-ecosystem publish mode, and `--dry-run` does not imply a default target

When a name is present:

1. Read the "Type Detection" section from publish-rules.md
2. Determine if the name is a skill or plugin using the Bash existence checks
3. Execute pipeline in execution order: Stages 1, 2, 0, 3, 4, 5, 6, 7, 8
4. For read-only stages (1, 2, 0, 3): run the stage body normally regardless of MODE — these stages do not touch disk
5. For write stages (4, 5, 6, 7, 8):
   - If MODE is `execute`: use the **EXECUTE behaviour** block in each stage
   - If MODE is `dry-run`: use the **DRY-RUN behaviour** block in each stage (per D-12 contract in publish-rules.md Output Format Contracts section)
6. Print each stage's tick-mark line immediately as it completes:
   - MODE `execute`: D-02 format (`✅ <stage message>`)
   - MODE `dry-run`: D-12 format (`[DRY RUN] ✅ Stage N: <name> — would <verb phrase>`)
7. If MODE is `dry-run`, the D-12 summary table is the LAST output of the command on EVERY exit path:
   - On success: after the `✔ [DRY RUN] Simulated ...` line
   - On hard-stop failure at any stage: after the `[DRY RUN] ❌` failure block
   - On warning-abort (PREFLIGHT 0b/0c answered N): after the abort line
   See Scenario E in publish-rules.md for the authoritative exit-path contract

## Pipeline Flow

Execute these stages in sequence. Each stage must succeed before the next begins.

Each write stage (4–8) has both an **EXECUTE behaviour** block and a **DRY-RUN behaviour** block in publish-rules.md. The MODE variable from Input Handling selects which block runs. Read-only stages (1, 2, 0, 3) have no branch — they run the same in both modes because they never touch disk.

1. **Validation Gate** — run all validation checks from validation-rules.md; hard stop and show full report if any check fails (D-09)
2. **Version Bump Prompt** — read current version, compute three bump options, present the D-04 prompt; wait for user to select 1, 2, or 3
3. **PREFLIGHT** — run 4 environment checks (semver validity, version regression, remote sync, manifest presence and name match); hard stop on 0a/0d failure, warning on 0b/0c (D-11)
4. **Changelog Summary Prompt** — present "Describe the change in one sentence:"; wait for user response (D-05)
5. **Write Version Bump** — update version field in SKILL.md frontmatter (skills) or plugin.json (plugins)
6. **Write CHANGELOG Entry** — insert new entry at top of CHANGELOG.md in container directory
7. **Package** — ZIP for skills (cd into claude-code-skill/ first to ensure SKILL.md is at root); rsync for plugins (with pre-rsync .git cleanup)
8. **Copy to Marketplace + Update Index** — copy packaged artifact; update marketplace.json for plugins only; regenerate README.md for all artifacts
9. **Git Commit and Push** — stage all changes, show pre-push confirmation gate (y/N) with commit preview and staged file count; on user confirm: commit and push to origin/main; on push failure: attempt one auto pull-rebase recovery then retry (per D-10 two-tier system); on success: print verification nudge with GitHub URL (per D-13)

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
- If git push fails, attempt exactly one auto pull-rebase recovery before showing manual recovery (D-10 two-tier system); if rebase conflicts, run `git rebase --abort` immediately — never leave the marketplace repo mid-rebase
- The pre-push confirmation gate (y/N) is non-skippable — no `--force`, `--yes`, or bypass flag exists (per D-13)
- PREFLIGHT is non-skippable — always runs after version prompt, before changelog prompt
- In dry-run mode (`--dry-run`), NO files are created, modified, or pushed — every write stage emits a `[DRY RUN]` line describing what it would do instead; see D-12 in publish-rules.md for the exact contract
- In dry-run mode, Stage 8 emits zero git invocations — no `git add`, `git commit`, `git push`, and no git subshell calls to resolve branch or remote state (enforced by the Stage 8 DRY-RUN behaviour block in publish-rules.md, per the D-12 contract)
