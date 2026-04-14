# Changelog

## v0.3.0 — 2026-04-14

Publish pipeline now auto-updates both the per-plugin README version line and the marketplace index README on every publish, with skip-if-unchanged guards and a diff preview before the push confirmation gate. Also improves rsync hygiene by excluding common IDE config directories.

## v0.2.1 — 2026-04-13

Rename plugin to `marketplace-manager` and update command prefix. Fix ecosystem paths.

## v0.2.0 — 2026-04-02

Add natural language interface — describe distribution intents in plain English and the skill routes to the right command.

## v0.1.0 — 2026-04-01

Initial release. Six commands: `validate`, `new-skill`, `new-plugin`, `status`, `publish`, and a natural language routing skill.
