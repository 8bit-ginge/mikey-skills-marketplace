<!-- generated-by: gsd-doc-writer -->
# Testing

`marketplace-manager` is an instruction-based Claude Code plugin — all command logic lives in Markdown files that Claude interprets at runtime. There is no compiled code, no test framework, and no `package.json`. Testing is a combination of **structural grep assertions**, a **shell verification harness**, and **manual observation** in Claude Code.

---

## Test Approach

| Layer | What it covers | How it runs |
|-------|---------------|-------------|
| Structural greps | Key contracts exist in the right files (D-12 prefix contract, DRY-RUN behaviour blocks, two-tier recovery wording) | Bash one-liners — run in ~2 seconds |
| Shell harness | Dry-run mode writes nothing to disk or git state | `verify-dry-run-no-writes.sh` |
| Manual UAT | Live pipeline behaviour in Claude Code: prefix rendering, version prompt, summary table, preflight failures | Run `/marketplace-manager:publish` or `/marketplace-manager:validate` in Claude Code |

No test framework (`jest`, `vitest`, `pytest`, etc.) is installed or needed. All verification is either grep-based file content inspection or empirical shell snapshots.

---

## Structural Grep Assertions

These are the canonical per-phase quick checks. Run them from the plugin root directory.

### Core publish pipeline

```bash
# --dry-run flag is parsed by SKILL.md
grep -q "\-\-dry-run" skills/publish/SKILL.md && echo "PASS" || echo "FAIL"

# All write stages (4–8) have DRY-RUN behaviour blocks (expect >= 9)
count=$(grep -c "DRY-RUN behaviour" skills/publish/references/publish-rules.md)
[ "$count" -ge 9 ] && echo "PASS: $count blocks" || echo "FAIL: only $count blocks"

# D-12 output contract is documented
grep -q "### Dry-Run Output (D-12)" skills/publish/references/publish-rules.md && echo "PASS" || echo "FAIL"

# Two-tier push failure recovery is present
grep -q "two-tier system" skills/publish/SKILL.md && echo "PASS" || echo "FAIL"

# Pre-push confirmation gate reference is present
grep -q "per D-13" skills/publish/SKILL.md && echo "PASS" || echo "FAIL"
```

### Validation rules

```bash
# Validation rules reference file exists
[ -f "skills/validate/references/validation-rules.md" ] && echo "PASS" || echo "FAIL"

# Four check categories are defined
grep -q "Category 1: Container Structure" skills/validate/references/validation-rules.md && echo "PASS" || echo "FAIL"
```

### CHANGELOG and version checks

```bash
# publish-rules.md defines the changelog entry format
grep -q "CHANGELOG" skills/publish/references/publish-rules.md && echo "PASS" || echo "FAIL"
```

---

## Dry-Run Verification Harness

Phase 8 introduced a shell script that proves dry-run mode produces zero filesystem or git state changes. The script has three modes.

**Script location:** `.planning/milestones/v1.1-phases/08-dry-run-mode/scripts/verify-dry-run-no-writes.sh`

### Structural-only mode (no marketplace required)

Runs only the Markdown content assertions — does not require the marketplace repo to be present on disk.

```bash
bash .planning/milestones/v1.1-phases/08-dry-run-mode/scripts/verify-dry-run-no-writes.sh structural-only
```

Expected output:
```
PASS: 9 DRY-RUN behaviour blocks present
PASS: Stage 8 DRY-RUN block has zero executable git invocations
PASS: structural checks
```

Exit code `0` means all structural assertions pass.

### Full empirical mode (requires marketplace repo)

Used when the marketplace repo is present at the path defined inside the script. This mode takes a before-snapshot, runs the dry-run, then verifies nothing changed.

```bash
# Step 1 — capture state
bash .planning/milestones/v1.1-phases/08-dry-run-mode/scripts/verify-dry-run-no-writes.sh snapshot

# Step 2 — run in Claude Code
# /marketplace-manager:publish marketplace-manager --dry-run
# (select any bump option, provide any summary string)

# Step 3 — verify nothing changed
bash .planning/milestones/v1.1-phases/08-dry-run-mode/scripts/verify-dry-run-no-writes.sh verify
```

Exit code `0` = no filesystem or git changes. Exit code `1` = something was written. Exit code `2` = invariant failure (missing directory).

> Note: The full empirical mode requires the marketplace repo at the path configured inside the script. If the repo is not present, use `structural-only` mode — it covers the critical safety assertions.

---

## Manual Observation Tests

These tests require running commands in Claude Code and observing the output. They cannot be automated because they depend on Claude's runtime behaviour.

### Validate command

| Test | How to run | What to check |
|------|-----------|---------------|
| Single artifact validation | `/marketplace-manager:validate <name>` | Report shows Container, SKILL.md Frontmatter (or Plugin JSON), and Documentation sections; each check shows pass/fail with fix message on fail |
| Full ecosystem scan | `/marketplace-manager:validate` (no argument) | Discovers all skills and plugins; ends with summary table |
| Unknown artifact | `/marketplace-manager:validate nonexistent-thing` | Reports artifact not found — does not crash |

### Publish dry-run

| Test | How to run | What to check |
|------|-----------|---------------|
| Dry-run happy path | `/marketplace-manager:publish <name> --dry-run` | Every stage output line carries `[DRY RUN]` prefix; D-12 summary table appears as the last output; no files written |
| Version bump prompt | During dry-run, observe Stage 2 | Three semver options shown (patch, minor, major); pipeline waits for selection |
| Dry-run on preflight failure | Force a PREFLIGHT failure (e.g., enter an invalid semver at the prompt) | Pipeline stops; D-12 summary table still appears as the last output with failed stage marked FAIL and later stages marked SKIP |

### Publish execute mode

| Test | How to run | What to check |
|------|-----------|---------------|
| Validation gate blocks bad artifact | Run `/marketplace-manager:publish <artifact-with-failures>` | Pipeline stops at Stage 1 with full validation report; no version prompt appears |
| Pre-push confirmation gate | Run publish against a valid artifact | Stage 9 shows y/N prompt with commit message and staged file count; commit and push only proceed on `y` |
| Missing artifact name | `/marketplace-manager:publish --dry-run` (name omitted) | Prints usage line; stops — does not attempt to publish |

### Status command

| Test | How to run | What to check |
|------|-----------|---------------|
| Full mode | `/marketplace-manager:status` | Cards with git divergence footer and summary footer appear for all skills and plugins |
| Brief inventory | `/marketplace-manager:status --brief` | Heading `## marketplace: status (brief)`; `### Skills (N)` and `### Plugins (M)` sections; one `- <name>@<version>` line per artifact; alphabetically sorted; closes with `Run without --brief for sync state and marketplace comparison.` |
| List alias | `/marketplace-manager:status --list` | Output identical to `--brief` mode |

### New-plugin scaffolding

| Test | How to run | What to check |
|------|-----------|---------------|
| Scaffold smoke test | `/marketplace-manager:new-plugin <name>` | Plugin directory created; both `CHANGELOG.md` and `CHANGELOG-internal.md` land at the plugin root; silent validate prints `✔ validate: <name> — all checks passed` |

---

## Coverage Requirements

No coverage thresholds are configured — this project has no runtime code to measure coverage against.

The closest equivalent is ensuring every command has:
1. At least one structural grep assertion covering its key contract
2. A UAT test record in `.planning/milestones/v1.1-phases/<phase>/` covering happy path and at least one failure path

---

## CI Integration

No CI pipeline is configured for this plugin. All verification is run locally by the developer before committing.

The recommended pre-commit check is:

```bash
bash .planning/milestones/v1.1-phases/08-dry-run-mode/scripts/verify-dry-run-no-writes.sh structural-only
```

This exits `0` in under 5 seconds and catches the most critical structural regressions (DRY-RUN behaviour block count, Stage 8 git isolation).
