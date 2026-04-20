# Phase Spec: Desktop Skill Sync

**Status:** Exploration
**Proposed phase:** 4.1 (inserted between Phase 4: Publish and Phase 5: NL Interface)
**Owner:** Mikey East
**Date:** 2026-04-20

---

## Problem

When `/marketplace:publish [name]` runs on a skill, it packages the `claude-code-skill/` directory as a ZIP and pushes it to the marketplace repo. But the local `claude-desktop-skill/<name>.skill` file — the installable for Claude Desktop — is never updated as part of this pipeline.

This creates drift: the marketplace gets the latest version, but the local `.skill` file stays stale. A developer installing from their local file gets an older version than what's live in the marketplace.

---

## Goal

Every skill published to the marketplace also writes an up-to-date `.skill` file to `claude-desktop-skill/` in the same operation — so local and marketplace are always in sync after a publish run.

---

## Proposed Requirements

| ID | Requirement |
|----|-------------|
| **DSYNC-01** | Publish writes an updated `claude-desktop-skill/<name>.skill` ZIP as part of the skill packaging step, before any marketplace writes occur |
| **DSYNC-02** | The `.skill` ZIP is built from `claude-code-skill/` using the same exclusion rules as marketplace packaging: excludes `evals/`, `*.pyc`, `__pycache__/`, `.DS_Store` |
| **DSYNC-03** | `SKILL.md` sits at the root of the ZIP (not nested under `claude-code-skill/`) — matching Claude Desktop's expected install structure |
| **DSYNC-04** | After writing, publish verifies the `.skill` file with `unzip -l` and reports the file size and entry count |
| **DSYNC-05** | If `claude-desktop-skill/` does not exist, publish creates it with `mkdir -p` before writing |
| **DSYNC-06** | The `.skill` file path is included in the publish summary output so the developer knows it was updated |
| **DSYNC-07** | Desktop sync failure is a hard stop — publish does not push to the marketplace if the local `.skill` file could not be written |

---

## Publish Pipeline — Revised Step Order

Current Phase 4 pipeline:
```
validate → version bump → changelog → package → copy to marketplace → index → git commit + push
```

Proposed with desktop sync inserted:
```
validate → version bump → changelog → package → write .skill to claude-desktop-skill/ → copy to marketplace → index → git commit + push
```

Desktop sync happens **after** packaging (step 4) but **before** any marketplace writes (step 5) — so if `.skill` creation fails, no remote state is touched.

---

## Success Criteria

1. After running `/marketplace:publish interview-research`, `claude-desktop-skill/interview-research.skill` is updated and its internal `SKILL.md` matches the version just published
2. `unzip -l claude-desktop-skill/interview-research.skill` shows `SKILL.md` at the root (not `claude-code-skill/SKILL.md`)
3. If the `claude-desktop-skill/` directory does not exist, it is created automatically — no manual setup needed
4. The publish summary explicitly states the `.skill` file path and byte size, e.g. `✓ Desktop: claude-desktop-skill/interview-research.skill (14.2 KB)`
5. A missing or unwritable `claude-desktop-skill/` path stops the pipeline before any git writes with a clear error message

---

## Integration Notes

- **Relation to `pack` skill:** `pack` is the manual, standalone way to create a `.skill` file. Desktop sync in publish makes `pack` optional for standard releases — the developer only needs `pack` for ad-hoc packaging outside the publish flow.
- **Relation to VAL-03:** The validate phase already checks that `claude-desktop-skill/` exists as a directory. Post-publish, validate will now also be able to implicitly verify the `.skill` file is present and non-stale (future v2 requirement opportunity).
- **Relation to PUB-04:** PUB-04 describes the ZIP packaging step. DSYNC-01 through DSYNC-04 extend that step with a second output target — same source, different destination.

---

## Out of Scope

| Feature | Reason |
|---------|--------|
| Auto-installing to Claude Desktop | No API for programmatic skill install — user must upload manually via Settings → Capabilities → Skills |
| Verifying the `.skill` file opens correctly in Claude Desktop | No headless verification path available |
| Syncing `.skill` files for plugins | Plugins distribute differently — no Claude Desktop `.skill` format for plugins |
| Updating a previously uploaded skill in Claude Desktop automatically | Claude Desktop has no CLI or API for remote skill management |

---

## Open Questions

1. Should the `.skill` file be committed to the marketplace repo alongside the marketplace ZIP, or remain local-only? (Current assumption: local-only — marketplace repo gets the extracted source, not the binary.)
2. Should `/marketplace:status` surface whether the local `.skill` file is out of date relative to the `claude-code-skill/` source? (Likely yes — natural v2 addition once DSYNC is in place.)
