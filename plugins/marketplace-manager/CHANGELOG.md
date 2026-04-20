# Changelog

## v0.3.6 — 2026-04-20

Pipeline hygiene: exclude .internal-documentation/ from marketplace rsync.

## v0.3.5 — 2026-04-20

Fix /status false-negatives: manifest-driven marketplace lookup with -plugin suffix resolution

## v0.3.4 — 2026-04-20

Stage 2a bump recommendation now ignores archival two-segment milestone tags — only v publish tags anchor the baseline.

## v0.3.3 — 2026-04-19

Fix /status rendering real skill versions — no more phantom @MISSING rows for valid skills.

<!--
  User-facing changelog — one short sentence per release.
  Syncs to the marketplace copy on every publish (Stage 6 rsync).
  Dev history (milestones, phases, decisions) lives in CHANGELOG-internal.md.
-->

## [Unreleased]

## v0.3.2 — 2026-04-19

Publish pipeline now recommends a bump type from commit history and separates user-facing and internal changelogs.

## v0.3.1 — 2026-04-15

Verify Stage 5 source-write fix end-to-end

## v0.3.0 — 2026-04-14

Auto-update READMEs on publish — per-plugin README version line (Phase 14) and marketplace index README with skip-if-unchanged guard and diff display before the push gate (Phase 15). Also fixes the rsync exclude list to drop IDE config directories `.vscode/`, `.idea/`, `.vs/` (Phase 16).

## v0.2.1 — 2026-04-13

Rename plugin to marketplace-manager, fix ecosystem paths, update command prefix

## v0.2.0 — 2026-04-02

Natural language skill detection and routing via marketplace-manager skill.

## v0.1.0 — 2026-04-01

Initial plugin structure with validate, scaffold, status, and publish commands.
