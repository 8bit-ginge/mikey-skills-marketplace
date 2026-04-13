# Changelog

All notable changes to the Marketplace Manager plugin.

Format follows [Keep a Changelog](https://keepachangelog.com/). Version numbers track `plugin.json`.

## v0.2.1 — 2026-04-13

Rename plugin to marketplace-manager, fix ecosystem paths, update command prefix

## [v1.1] - 2026-04-13

**Zero-Intervention Publish Pipeline with Guardrails** — 7 phases, 10 plans, 15 requirements satisfied.

### Added

- **PREFLIGHT stage (Stage 0)** — 4 environment checks that block before any writes begin:
  - PRE-01: Semver format validation (rejects malformed version strings)
  - PRE-02: Version regression guard (prevents publishing <= current version)
  - PRE-03: Remote sync check (detects if marketplace remote is ahead of local)
  - PRE-04: Plugin name / manifest name match (catches renames not propagated to plugin.json)
- **Dry-run mode** — `--dry-run` flag simulates the full 9-stage pipeline with no files written or pushed
  - Every output line prefixed with `[DRY RUN]` to distinguish simulation from real execution
  - Per-stage pass/fail output showing what each stage would do
  - Summary table at the end showing all stage results in one view
- **Execute mode auto-recovery** — automatic pull-rebase on push failure with `rebase --abort` safety net
- **Pre-push confirmation gate** — y/N prompt after all local writes, before the irreversible git push
- **Post-publish verification nudge** — prompts developer to verify update is visible on GitHub
- **Progressive checkmark output** — status line displayed immediately after each stage completes, across all modes

### Changed

- Pipeline expanded from 8 stages to 9 (PREFLIGHT added as Stage 0)
- Output contracts consolidated and named (D-01 through D-13) for traceable format specifications
- Per-stage DRY RUN / EXECUTE blocks replace document-level mode flag in publish-rules.md
- SKILL.md `argument-hint` updated to `[name] [--dry-run]`

### Fixed

- SKILL.md regression from Phase 11 tech debt cleanup (dry-run orchestration was inadvertently removed)
- Stale cross-references: stage count corrected (8 to 9), contract reference corrected (D-04 to D-13)
- SUMMARY frontmatter standardised with `requirements_completed` across all phases
- publish-rules.md verb-phrase table and stage block text consistency
- verify-dry-run-no-writes.sh missing `set -e` and `-o pipefail` shell safety flags

## [v1.0] - 2026-04-10

**MVP** — 6 phases, 12 plans. Plugin self-published to marketplace as proof the pipeline works end-to-end.

### Added

- `/marketplace-manager:validate [name]` — structured pass/fail checks against Anthropic spec and project conventions, with fix guidance for every failure
- `/marketplace-manager:new-skill [name]` — scaffold a skill container that passes validation immediately on creation
- `/marketplace-manager:new-plugin [name]` — scaffold a plugin container that passes validation immediately on creation
- `/marketplace-manager:status` — three-source sync table (local dev, marketplace repo, Claude Code install cache) with drift detection
- `/marketplace-manager:publish [name]` — 8-stage gated pipeline: validate, version prompt, version bump, changelog, package (zip + rsync), marketplace.json update, README index regeneration, git push
- `/marketplace-manager:marketplace-manager` — natural language interface routing 5 distribution intents to existing command workflows
- `.claude-plugin/plugin.json` manifest for Claude Code plugin recognition
- Thin orchestration pattern: SKILL.md bodies under 70 lines, all logic in reference files (validation-rules.md, scaffold-templates.md, status-rules.md, publish-rules.md)

### Fixed

- Hardcoded user paths replaced with `${CLAUDE_PLUGIN_ROOT}` across 59 locations in 5 files (Phase 6)
- Semver normalisation added after publish UAT found edge cases

## [v0.2.0] - 2026-04-02

### Added

- Natural language skill detection and routing via marketplace-manager skill

## [v0.1.0] - 2026-04-01

### Added

- Initial plugin structure with validate, scaffold, status, and publish commands

[v1.1]: https://github.com/8bit-ginge/marketplace-manager/compare/v1.0...v1.1
[v1.0]: https://github.com/8bit-ginge/marketplace-manager/releases/tag/v1.0
