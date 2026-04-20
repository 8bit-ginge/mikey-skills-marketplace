# Publish Rules

Complete pipeline logic for the `/marketplace-manager:publish` command. This is the single source of truth for all stage definitions, ecosystem paths, packaging commands, output contracts, and failure handling. SKILL.md loads this file via explicit Read instruction and delegates all execution here.

## Table of Contents

1. [Ecosystem Paths](#ecosystem-paths)
2. [Type Detection](#type-detection)
3. [Pipeline Overview](#pipeline-overview)
4. [Stage 1: Validation Gate](#stage-1-validation-gate)
5. [Stage 2: Version Bump Prompt](#stage-2-version-bump-prompt)
6. [Stage 0: PREFLIGHT](#stage-0-preflight)
7. [Stage 3: Changelog Summary Prompt](#stage-3-changelog-summary-prompt)
8. [Stage 4: Write Version Bump](#stage-4-write-version-bump)
9. [Stage 5: Write CHANGELOG Entry](#stage-5-write-changelog-entry)
10. [CHANGELOG Conventions](#changelog-conventions)
11. [Stage 6: Package](#stage-6-package)
12. [Stage 7: Copy to Marketplace and Update Index](#stage-7-copy-to-marketplace-and-update-index)
13. [Stage 8: Git Commit and Push](#stage-8-git-commit-and-push)
14. [Output Format Contracts](#output-format-contracts)
    - [Dry-Run Output (D-12)](#dry-run-output-d-12)
15. [Failure Handling](#failure-handling)

---

## Ecosystem Paths

These absolute paths are the ground truth for locating skills, plugins, and the marketplace. Do NOT embed these in SKILL.md body — they live here only.

- **Ecosystem root:** `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/`
- **Skills directory:** `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/`
- **Plugins directory:** `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/`
- **Marketplace repo:** `/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/`
- **Marketplace remote:** `https://github.com/8bit-ginge/mikey-skills-marketplace.git`

Abbreviated references used throughout this file:
- `<skills-dir>` = `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/`
- `<plugins-dir>` = `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/`
- `<marketplace>` = `/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/`

---

## Type Detection

Given a name from `$ARGUMENTS`, determine whether it is a skill or plugin. Use the same logic as validation-rules.md.

1. Check if `<skills-dir>/<name>/` exists as a directory — if yes, type is `skill`
2. Check if `<plugins-dir>/<name>/` exists as a directory — if yes, type is `plugin`
3. If both exist, treat as `skill` (skills take precedence)
4. If neither exists, report: `Artifact '<name>' not found in skills/ or plugins/` and stop

```bash
[ -d "/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/<name>" ] && echo "skill"
[ -d "/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/<name>" ] && echo "plugin"
```

---

## Pipeline Overview

Nine sequential stages. Each stage is gated — no stage starts until the previous succeeds. If any stage fails, the pipeline stops immediately and shows the failure output defined in the Failure Handling section.

| Stage | Description | Writes? |
|-------|-------------|---------|
| 1 | Validation Gate | No |
| 2 | Version Bump Prompt | No |
| 0 | PREFLIGHT (4 checks) | No |
| 3 | Changelog Summary Prompt | No |
| 4 | Write Version Bump | Yes |
| 5 | Write CHANGELOG Entry | Yes |
| 6 | Package | Yes |
| 7 | Copy to Marketplace + Update Index | Yes |
| 8 | Git Commit and Push | Yes |

**Stages 1, 2, 0, and 3 are read-only.** Stages 4-8 write files. The validation gate (Stage 1) ensures no writes occur if the artifact has issues. The two prompts (Stages 2-3) collect all user input before any writes begin. PREFLIGHT (Stage 0) runs between the version prompt and the changelog prompt because checks 0a and 0b require the proposed version from Stage 2.

**Dry-run output prefix rule (D-12 cross-cutting contract).** When MODE=dry-run, every output line emitted by ANY stage is prefixed with `[DRY RUN] ` per D-12, regardless of whether the stage has a dedicated `**DRY-RUN behaviour:**` block. Read-only stages (Stage 1 Validation, Stage 2 Version Prompt, Stage 0 PREFLIGHT, Stage 3 Changelog Prompt) execute their normal body in dry-run mode, but every line they print — including check rows, delimiter lines, warning-continue prompts (`Continue anyway? (y/N)`), and hard-stop failure blocks — carries the `[DRY RUN] ` prefix. Write stages (4–8) additionally follow their `**DRY-RUN behaviour:**` block instead of their `**EXECUTE behaviour:**` block. This rule is the single source of truth for prefix coverage: if a line comes out of the publish pipeline in dry-run mode, it has `[DRY RUN] ` on the front.

---

## Stage 1: Validation Gate

**Requirement:** PUB-02 — hard stop if any checks fail before any writes occur.

Run the full validation logic from `validation-rules.md` against the artifact. To do this:

1. Read `${CLAUDE_PLUGIN_ROOT}/skills/validate/references/validation-rules.md` to load all check definitions
2. Execute ALL applicable check categories for the detected artifact type:
   - For skills: Container Structure, SKILL.md Frontmatter, Documentation
   - For plugins: Container Structure, Plugin JSON, Documentation
3. Tally the total number of failures

**If ANY check fails:**
- Stop the pipeline immediately — do NOT proceed to Stage 2
- Display the full validation report in the standard D-03 format (same output as `/marketplace-manager:validate`)
- After the report, append:
  ```
  Fix the issues above, then run `/marketplace-manager:publish <name>` again.
  ```
- Do NOT proceed to any write stage

**If all checks pass:**
- Print: `✅ Validation passed`
- Continue to Stage 2

---

## Stage 2: Version Bump (Recommendation + Prompt)

Stage 2 has two sub-steps: **2a** computes an advisory bump recommendation from conventional commits on the artifact's source repo; **2b** presents the interactive bump-type prompt. 2a is advisory only — the developer still chooses freely.

### Sub-step 2a: Bump Recommendation

Read-only: scans the **artifact's source repo only** (never the marketplace repo) for conventional commits since the last reachable tag and prints `Recommend: <type> — <rationale>`. The developer still chooses freely at sub-step 2b — this is advisory (per D-02, D-05, BUMP-01, BUMP-02).

**EXECUTE:**

```bash
SOURCE_DIR="<source-dir>"  # e.g. /Users/<user>/.../plugins/<name>/ — container_dir, same derivation as Stage 5

# D-07 branch: no source git repo → skip recommendation, still allow prompt at 2b
if ! git -C "$SOURCE_DIR" rev-parse --show-toplevel >/dev/null 2>&1; then
  echo "Recommend: (none) — no git history available for <name>"
else
  # D-03 baseline / D-06 fallback: resolve tag range
  # D-03/D-06 baseline — restrict to three-segment semver publish tags (Stage 8a2 shape).
  # Archival two-segment milestone tags (v1.0, v1.1, v1.2) must NOT anchor the bump recommendation.
  if LAST_TAG=$(git -C "$SOURCE_DIR" describe --tags --abbrev=0 --match 'v[0-9]*.[0-9]*.[0-9]*' 2>/dev/null); then
    RANGE_DESC="since $LAST_TAG"
    COMMIT_BODY=$(git -C "$SOURCE_DIR" log "$LAST_TAG"..HEAD --format=%B)
  else
    RANGE_DESC="no prior tag; scanned last 50 commits"
    COMMIT_BODY=$(git -C "$SOURCE_DIR" log -50 --format=%B)
  fi

  # D-04 heuristic applied via stdlib Python
  echo "$COMMIT_BODY" | python3 -c "
import sys, re
body = sys.stdin.read()
range_desc = '$RANGE_DESC'

# D-08 degenerate case
if not body.strip():
    print(f'Recommend: patch — no commits found {range_desc}')
    sys.exit(0)

# D-04 rule 1: major (BREAKING CHANGE footer OR type! prefix)
has_breaking = bool(re.search(r'^BREAKING CHANGE', body, re.MULTILINE))
has_bang     = bool(re.search(r'^[a-z]+(\([^)]+\))?!:', body, re.MULTILINE))

# Counts for rationale detail
feat_count = len(re.findall(r'^feat(\(|:)', body, re.MULTILINE))
fix_count  = len(re.findall(r'^fix(\(|:)',  body, re.MULTILINE))

if has_breaking or has_bang:
    signal = 'BREAKING CHANGE' if has_breaking else \"'!' marker on type\"
    print(f'Recommend: major — {signal} {range_desc}')
elif feat_count > 0:
    print(f'Recommend: minor — {feat_count} feat commit(s) {range_desc}, no breaking changes')
else:
    print(f'Recommend: patch — {fix_count} fix commit(s) {range_desc}')
"
fi
```

Print a progress checkmark after the recommendation line: `echo "✅ Stage 2a: Bump Recommendation computed"` (per CLAUDE.md "No silent stages" rule — the `Recommend: …` line itself is the visible progress).

**DRY RUN:**

Per D-19, Stage 2a runs the same computation as EXECUTE (read-only `git` reads are safe in dry-run — same contract as PREFLIGHT 0c at publish-rules.md:246-258). Only the `print(...)` statements gain the `[DRY RUN] ` prefix. The verbatim DRY RUN block, modeled after PREFLIGHT 0c, prints one of four output lines depending on source-repo state:

```bash
SOURCE_DIR="<source-dir>"  # same derivation as EXECUTE

# D-07 branch: no source git repo
if ! git -C "$SOURCE_DIR" rev-parse --show-toplevel >/dev/null 2>&1; then
  echo "[DRY RUN] ✅ Stage 2a: Bump Recommendation — Recommend: (none) — no git history available for <name>"
else
  # D-03 baseline / D-06 fallback: resolve tag range (read-only)
  # D-03/D-06 baseline — restrict to three-segment semver publish tags (Stage 8a2 shape).
  # Archival two-segment milestone tags (v1.0, v1.1, v1.2) must NOT anchor the bump recommendation.
  if LAST_TAG=$(git -C "$SOURCE_DIR" describe --tags --abbrev=0 --match 'v[0-9]*.[0-9]*.[0-9]*' 2>/dev/null); then
    RANGE_DESC="since $LAST_TAG"
    COMMIT_BODY=$(git -C "$SOURCE_DIR" log "$LAST_TAG"..HEAD --format=%B)
  else
    RANGE_DESC="no prior tag; scanned last 50 commits"
    LAST_TAG=""
    COMMIT_BODY=$(git -C "$SOURCE_DIR" log -50 --format=%B)
  fi

  # D-04 heuristic applied via stdlib Python — same body as EXECUTE; only print lines gain [DRY RUN] prefix
  echo "$COMMIT_BODY" | python3 -c "
import sys, re
body = sys.stdin.read()
range_desc = '$RANGE_DESC'
last_tag = '$LAST_TAG'

# D-08 degenerate case
if not body.strip():
    print(f'[DRY RUN] ✅ Stage 2a: Bump Recommendation — Recommend: patch — no commits found since {last_tag}')
    sys.exit(0)

has_breaking = bool(re.search(r'^BREAKING CHANGE', body, re.MULTILINE))
has_bang     = bool(re.search(r'^[a-z]+(\([^)]+\))?!:', body, re.MULTILINE))
feat_count = len(re.findall(r'^feat(\(|:)', body, re.MULTILINE))
fix_count  = len(re.findall(r'^fix(\(|:)',  body, re.MULTILINE))

if has_breaking or has_bang:
    signal = 'BREAKING CHANGE' if has_breaking else \"'!' marker on type\"
    print(f'[DRY RUN] ✅ Stage 2a: Bump Recommendation — Recommend: major — {signal} {range_desc}')
elif feat_count > 0:
    print(f'[DRY RUN] ✅ Stage 2a: Bump Recommendation — Recommend: minor — {feat_count} feat commit(s) {range_desc}, no breaking changes')
else:
    if last_tag:
        print(f'[DRY RUN] ✅ Stage 2a: Bump Recommendation — Recommend: patch — {fix_count} fix commit(s) {range_desc}')
    else:
        print(f'[DRY RUN] ✅ Stage 2a: Bump Recommendation — Recommend: {(\"patch\" if fix_count == 0 else \"patch\")} — no prior tag; scanned last 50 commits, {fix_count} fix commit(s)')
"
fi
```

All four output paths must be reachable by inspection of the DRY RUN block:

- **Normal path** (tag exists, commits found):
  `[DRY RUN] ✅ Stage 2a: Bump Recommendation — Recommend: <type> — <rationale>`
- **D-07 path** (no source `.git`):
  `[DRY RUN] ✅ Stage 2a: Bump Recommendation — Recommend: (none) — no git history available for <name>`
- **D-08 path** (no commits since last tag):
  `[DRY RUN] ✅ Stage 2a: Bump Recommendation — Recommend: patch — no commits found since <last-tag>`
- **Fallback path** (no prior tag, scans last 50 commits):
  `[DRY RUN] ✅ Stage 2a: Bump Recommendation — Recommend: <type> — no prior tag; scanned last 50 commits, <signal>`

Note: the `git log` / `git describe` reads are strictly read-only and safe in dry-run — same contract as PREFLIGHT 0c (see publish-rules.md:246-258).

**Rationale line shapes (D-05):**

- `Recommend: major — BREAKING CHANGE since v0.3.1`
- `Recommend: major — '!' marker on type since v0.3.1`
- `Recommend: minor — 3 feat commit(s) since v0.3.1, no breaking changes`
- `Recommend: patch — 5 fix commit(s) since v0.3.1`
- `Recommend: patch — no commits found since v0.3.1`
- `Recommend: patch — 0 fix commit(s) no prior tag; scanned last 50 commits`
- `Recommend: (none) — no git history available for <name>`

### Sub-step 2b: Version Bump Prompt

**Requirement:** PUB-03 — present version bump options before any writes.

#### Step 1: Read current version

**For skills** — read from `<skills-dir>/<name>/claude-code-skill/SKILL.md` YAML frontmatter:

```bash
python3 -c "
import re, sys
content = open('/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/<name>/claude-code-skill/SKILL.md').read()
m = re.search(r'^---\n(.*?)^---', content, re.MULTILINE | re.DOTALL)
if m:
    vm = re.search(r'^version:\s*(.+)$', m.group(1), re.MULTILINE)
    print(vm.group(1).strip().lstrip('v') if vm else 'MISSING')
else:
    print('MISSING')
"
```

**For plugins** — read from `<plugins-dir>/<name>/.claude-plugin/plugin.json`:

```bash
python3 -c "
import json
d = json.load(open('/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/<name>/.claude-plugin/plugin.json'))
print(d.get('version', 'MISSING'))
"
```

#### Step 2: Normalize version to semver and compute bump options

**All versions are normalized to semver (X.Y.Z) before computing bumps.** If the current version has no dots (e.g., `2`), treat it as `2.0.0`. If it has one dot (e.g., `2.1`), treat it as `2.1.0`. This ensures patch, minor, and major bumps always produce different results.

```python
version = '<current version>'

# Normalize to semver
parts = [int(x) for x in version.split('.')]
while len(parts) < 3:
    parts.append(0)

# Compute three distinct bump options
patch_ver = f'{parts[0]}.{parts[1]}.{parts[2]+1}'
minor_ver = f'{parts[0]}.{parts[1]+1}.0'
major_ver = f'{parts[0]+1}.0.0'
```

#### Step 3: Present the prompt (D-04 format)

```
Current version: <version>

What kind of change is this?
1. Patch (bug fix) → <patch_ver>
2. Minor (new feature) → <minor_ver>
3. Major (breaking change) → <major_ver>
```

Wait for user response (1, 2, or 3). Store the selected new version as `<new-version>`.

**The selected version is always in semver format** (e.g., `2.0.1`), regardless of the original format. This aligns with the project convention that versions follow `X.X.X`.

**DRY RUN:**

Stage 2b is a read-only interactive prompt; in dry-run mode the pipeline does NOT block for user input. Print exactly one line (per D-19 and the D-12 per-stage verb-phrase contract):

```
[DRY RUN] ✅ Stage 2b: Version Bump Prompt — would prompt for bump type (selected: <type> → <new-version>)
```

Per D-19, `<type>` defaults to the recommendation computed by Sub-step 2a for dry-run display purposes (patch / minor / major / none), and `<new-version>` is the version that would result from applying that bump to `<current-version>`.

---

## Stage 0: PREFLIGHT

**Purpose:** Catch environment problems before any file writes begin. PREFLIGHT runs after Stage 2 (version bump prompt) because checks 0a and 0b require the proposed new version as input. All four checks are grouped here as a single cohesive preflight block.

**Position in pipeline:** Stage 1 (validate) → Stage 2 (version prompt) → **PREFLIGHT** → Stage 3 (changelog prompt) → Stages 4-8 (writes).

Print `--- Preflight ---` before the first check.

### Check 0a: Semver Format Validity

**Severity:** Hard stop (per D-05)

Validate the proposed new version from Stage 2 is valid semver (X.Y.Z):

```bash
echo "$PROPOSED_VER" | grep -qE '^[0-9]+\.[0-9]+\.[0-9]+$'
```

**On pass:** Print `✅ Semver: <proposed-version> is valid`

**On fail:** Print:
```
❌ Semver: '<proposed-version>' is not valid semver (expected X.Y.Z)

**What happened:** The version string is not in X.Y.Z format. Only digits and dots are allowed (e.g., 1.2.3).

**Your changes are safe** — no files have been written. Nothing was lost.

**To fix:** Run `/marketplace-manager:publish <name>` again and enter a valid version when prompted.
```
Stop the pipeline immediately. Do NOT proceed to check 0b.

### Check 0b: Version Regression Guard

**Severity:** Warning with continue prompt (per D-07)

Compare the proposed new version against the current version:

```python
python3 -c "
def parse_semver(v):
    parts = [int(x) for x in v.split('.')]
    while len(parts) < 3:
        parts.append(0)
    return tuple(parts)

current = '<current-version>'
proposed = '<proposed-version>'
c = parse_semver(current)
p = parse_semver(proposed)

if p <= c:
    print('WARN')
else:
    print('PASS')
"
```

**On PASS:** Print `✅ Version: <proposed-version> > current <current-version>`

**On WARN:** Print:
```
⚠️  Version: <proposed-version> is not greater than <current-version> — Continue anyway? (y/N)
```
If user enters N or empty, stop the pipeline. If user enters Y, continue to check 0c.

### Check 0c: Remote Sync Check

**Severity:** Warning, no hard stop (per D-08)

Check if the marketplace local branch is behind remote. Guard for first-ever push where origin/main may not exist:

```bash
MARKETPLACE="/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace"

if git -C "$MARKETPLACE" ls-remote origin main 2>/dev/null | grep -q main; then
  BEHIND=$(git -C "$MARKETPLACE" rev-list HEAD..origin/main --count 2>/dev/null || echo "0")
  if [ "$BEHIND" -gt 0 ]; then
    echo "WARN"
  else
    echo "PASS"
  fi
else
  echo "PASS_FIRST_PUSH"
fi
```

**On PASS:** Print `✅ Remote: no commits ahead of local`

**On PASS_FIRST_PUSH:** Print `✅ Remote: first publish — remote branch not yet established`

**On WARN:** Print:
```
⚠️  Remote: marketplace remote has <count> commit(s) ahead of local — push may fail
    Suggested fix: git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace pull --rebase origin main
```
Continue regardless — user may know the remote state is acceptable. Phase 9 adds auto-recovery here.

### Check 0d: Manifest Presence Check

**Severity:** Hard stop (per D-06)

Confirm the artifact has a valid manifest file before any writes occur.

**For plugins:** Check `.claude-plugin/plugin.json` exists, is parseable JSON, and its `name` field matches the plugin directory name:

```bash
if [ ! -f "<plugins-dir>/<name>/.claude-plugin/plugin.json" ]; then
  echo "FAIL_MISSING"
else
  MANIFEST_NAME=$(python3 -c "
import json, sys
try:
    d = json.load(open('<plugins-dir>/<name>/.claude-plugin/plugin.json'))
    print(d.get('name', ''))
except:
    print('FAIL_INVALID')
    sys.exit(1)
" 2>/dev/null)
  if [ $? -ne 0 ] || [ "$MANIFEST_NAME" = "FAIL_INVALID" ]; then
    echo "FAIL_INVALID"
  elif [ "$MANIFEST_NAME" != "<name>" ]; then
    echo "FAIL_NAME_MISMATCH:$MANIFEST_NAME"
  else
    echo "PASS"
  fi
fi
```

**For skills:** Check `claude-code-skill/SKILL.md` exists and contains frontmatter delimiters:

```bash
if [ ! -f "<skills-dir>/<name>/claude-code-skill/SKILL.md" ]; then
  echo "FAIL_MISSING"
else
  head -1 "<skills-dir>/<name>/claude-code-skill/SKILL.md" | grep -q "^---$" && echo "PASS" || echo "FAIL_INVALID"
fi
```

**On PASS:** Print `✅ Manifest: plugin.json valid — name matches directory` (plugins) or `✅ Manifest: SKILL.md found and valid` (skills)

**On FAIL_NAME_MISMATCH:** Print:
```
❌ Manifest: name mismatch — directory is '<name>', plugin.json says '<manifest-name>'

**What happened:** The plugin directory is named '<name>' but plugin.json has "name": "<manifest-name>". This usually means the plugin was renamed without updating plugin.json, or vice versa.

**Your changes are safe** — no files have been written. Nothing was lost.

**To fix:** Update the "name" field in <plugins-dir>/<name>/.claude-plugin/plugin.json to "<name>", then run /marketplace-manager:publish <name> again.
```
Stop the pipeline immediately.

**On FAIL_MISSING or FAIL_INVALID:** Print:
```
❌ Manifest: <manifest-file> missing or invalid in '<name>'

**What happened:** The manifest file was not found or could not be parsed at the expected path. This usually means the artifact was renamed but the manifest was not moved, or the artifact was not scaffolded correctly.

**Your changes are safe** — no files have been written. Nothing was lost.

**To fix:** Ensure the manifest exists at:
  <expected-path>
Then run `/marketplace-manager:publish <name>` again.
```
Stop the pipeline immediately.

Print `--- Pipeline ---` after all four checks pass (or after the last non-fatal check completes).

Continue to Stage 3.

---

## Stage 3: Changelog Summary Prompt

**Requirement:** PUB-03 — second separate prompt before any writes.

Stage 3 captures the user-facing release summary. The prompt deliberately names the taboos (phase numbers, milestone labels, internal jargon) to prime clean input the first time — cheaper than rewriting a leaked entry after Stage 6 rsync. Empty input reprompts (D-10); whitespace and newlines are sanitised on input (D-11).

**EXECUTE:**

```bash
while :; do
  echo "Summarise this release in one sentence (user-facing — avoid phase numbers, milestone labels, and internal jargon):"
  read -r RAW_SUMMARY
  # D-11 sanitisation: strip trailing newlines, collapse internal newlines to spaces, trim surrounding whitespace.
  SUMMARY=$(printf '%s' "$RAW_SUMMARY" | python3 -c "
import sys, re
raw = sys.stdin.read()
cleaned = re.sub(r'\s*\n\s*', ' ', raw).strip()
print(cleaned)
")
  if [ -n "$SUMMARY" ]; then
    break
  fi
  echo "Summary cannot be empty — please provide one sentence."
done
echo "✅ Stage 3: Summary captured"
```

The captured value is stored in `$SUMMARY` and consumed by Stage 5 (CHANGELOG entry) and, in Plan 18-02, by Stage 8a2 (release tag).

Do NOT combine this prompt with the version bump prompt. Two separate interactions — one thing at a time (D-03).

**DRY RUN:**

Stage 3 is a read-only interactive prompt; in dry-run mode the pipeline does NOT execute the while-loop or read stdin. Print exactly one line (per D-20):

```
[DRY RUN] ✅ Stage 3: Changelog Summary Prompt — would prompt for user-facing summary (captured)
```

---

## Stage 4: Write Version Bump

**Requirement:** PUB-01 — write the new version to the source file.

**EXECUTE behaviour:**

### For skills — update SKILL.md frontmatter

Read the SKILL.md file, locate the `version:` line within the `---` delimited frontmatter block only, replace with new version. Use python3 for safe replacement.

```python
import re

skill_md_path = '/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/<name>/claude-code-skill/SKILL.md'
new_version = '<new-version>'

content = open(skill_md_path).read()

# Find the frontmatter block (between first --- and second ---)
def replace_version_in_frontmatter(content, new_version):
    # Match the opening ---, frontmatter content, closing ---
    m = re.match(r'^(---\n)(.*?)(^---)', content, re.MULTILINE | re.DOTALL)
    if not m:
        raise ValueError('No YAML frontmatter block found')

    prefix = m.group(1)
    frontmatter = m.group(2)
    suffix = content[m.end():]

    # Replace version line within frontmatter only
    new_frontmatter = re.sub(
        r'^version:\s*.+$',
        f'version: {new_version}',
        frontmatter,
        flags=re.MULTILINE
    )

    return prefix + new_frontmatter + '---' + suffix

new_content = replace_version_in_frontmatter(content, new_version)

with open(skill_md_path, 'w') as f:
    f.write(new_content)
```

Write back the modified content. Preserve all other frontmatter fields and all body content exactly.

**Version format rule:** Write the version as-is from the bump selection (no leading `v` prefix — the version field stores the bare number or semver string, e.g., `3` or `2.1.4`).

### For plugins — update plugin.json

```python
import json

plugin_json_path = '/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/<name>/.claude-plugin/plugin.json'
new_version = '<new-version>'

with open(plugin_json_path) as f:
    data = json.load(f)

data['version'] = new_version

with open(plugin_json_path, 'w') as f:
    json.dump(data, f, indent=2)
    f.write('\n')
```

Print this stage's output per the Output Format Contracts section. Print it now, not later:

`✅ Version bumped: <old-version> → <new-version>`

**DRY-RUN behaviour:**

Do NOT invoke any python3 write command. Do NOT open `plugin.json` or `SKILL.md` for writing. Do NOT call any file-mutation function.

Print exactly one line:

```
[DRY RUN] ✅ Stage 4: Version Bump — would write version <new-version> to <manifest-file>
```

Where:
- `<new-version>` is the value chosen by the user at Stage 2
- `<manifest-file>` resolves to `plugin.json` for plugins or `SKILL.md frontmatter` for skills

Return success. Continue to Stage 5.

---

## Stage 5: Write CHANGELOG Entry

**Requirement:** PUB-01 — insert new entry at the top of CHANGELOG.md.

**EXECUTE behaviour:**

### Locate the container directory

- For skills: `<skills-dir>/<name>/`
- For plugins: `<plugins-dir>/<name>/`

### Read the current date

```bash
python3 -c "from datetime import date; print(date.today().strftime('%Y-%m-%d'))"
```

### Insert new entry

New entry format (Keep-a-Changelog variant observed in ecosystem):

```markdown
## v<new-version> — <YYYY-MM-DD>

<one-sentence summary from Stage 3>
```

**Note:** Use `—` (em dash, U+2014) between version and date. Write `v` prefix on version in changelog headings — e.g., `## v3 — 2026-03-31` or `## v2.1.4 — 2026-03-31`.

### Placement procedure

`changelog_path` is derived from the already-located `container_dir` (see "Locate the container directory" above) so the ecosystem path-template drift class of bug (PIPE-05) cannot recur.

```python
# changelog_path is derived from container_dir located earlier in this stage
# (see "Locate the container directory"). This avoids hard-coded ecosystem path
# templates that drift when the repo layout changes (the PIPE-05 failure class).
from pathlib import Path

changelog_path = Path(container_dir) / 'CHANGELOG.md'
new_version = '<new-version>'
today = '<YYYY-MM-DD>'       # already computed earlier in this stage
summary = '<summary>'         # release summary from Stage 3

with open(changelog_path) as f:
    content = f.read()

new_entry = f'## v{new_version} \u2014 {today}\n\n{summary}\n\n'

# Insert after the # Changelog header line (and any following blank lines)
import re
# Match # Changelog header followed by optional blank lines
new_content = re.sub(
    r'^(# Changelog\n+)',
    r'\g<1>' + new_entry,
    content,
    count=1,
    flags=re.MULTILINE,
)

if new_content == content:
    raise ValueError(
        'CHANGELOG.md does not contain a "# Changelog" header — '
        'entry was not inserted. Add the header and retry.'
    )

with open(changelog_path, 'w') as f:
    f.write(new_content)

# --- Post-write assertion (PIPE-06) ---------------------------------------
# Re-read the source CHANGELOG and confirm the new heading landed.
# Abort BEFORE Stage 6 if not, so rsync cannot ship a stale CHANGELOG.
with open(changelog_path) as f:
    written = f.read()

expected_heading = f'## v{new_version} \u2014 {today}'

# Collect first ~5 non-blank lines that appear AFTER the '# Changelog' header.
post_header = re.split(r'^# Changelog\n+', written, maxsplit=1, flags=re.MULTILINE)
first_nonblank: list[str] = []
if len(post_header) == 2:
    for line in post_header[1].splitlines():
        if line.strip():
            first_nonblank.append(line)
        if len(first_nonblank) >= 5:
            break

if expected_heading not in first_nonblank:
    import sys
    sys.stderr.write('\n')
    sys.stderr.write('❌ Stage 5 post-write assertion FAILED\n')
    sys.stderr.write(f'   Expected path:    {changelog_path}\n')
    sys.stderr.write(f'   Expected heading: {expected_heading}\n')
    sys.stderr.write(f'   Actual first lines after "# Changelog":\n')
    if first_nonblank:
        for line in first_nonblank:
            sys.stderr.write(f'     {line}\n')
    else:
        sys.stderr.write('     (none — "# Changelog" header not found in written file)\n')
    sys.stderr.write('   Note: Stage 6 rsync was skipped. Source CHANGELOG write did not\n')
    sys.stderr.write('         land at the expected path. Fix Stage 5 path resolution\n')
    sys.stderr.write('         before retrying.\n')
    raise RuntimeError(
        'Stage 5 post-write assertion failed — source CHANGELOG missing new entry; '
        'aborting before Stage 6.'
    )
```

Print this stage's output per the Output Format Contracts section. Print it now, not later:

`✅ CHANGELOG.md updated`

**DRY-RUN behaviour:**

Do NOT open `CHANGELOG.md` for writing. Do NOT compute the insertion regex. Do NOT create a new CHANGELOG.md if one does not exist.

Print exactly one line:

```
[DRY RUN] ✅ Stage 5: CHANGELOG Entry — would prepend v<new-version> entry to <container>/CHANGELOG.md
```

Where `<container>` is:
- `<plugins-dir>/<name>` for plugins (see Ecosystem Paths)
- `<skills-dir>/<name>` for skills (see Ecosystem Paths)

Return success. Continue to Stage 6.

---

## CHANGELOG Conventions

Two files in every plugin / skill source repo. One publishes, one does not. One pipeline produces both — Stage 3 captures the user-facing summary, Stage 5 writes it to `CHANGELOG.md`, Stage 6 rsyncs that file only, Stage 8a2 tags it on the source repo.

### Files

- **CHANGELOG.md — user-facing.**
  Lives in the source repo root AND in the marketplace copy (via Stage 6 rsync).
  One short sentence per release. Keep-a-Changelog heading shape (`## v<version> — <YYYY-MM-DD>`).
  Target audience: developers installing the plugin who want to know what's new.

- **CHANGELOG-internal.md — source-only.**
  Lives in the source repo root; NEVER rsyncs to the marketplace (Stage 6 `--exclude` enforces this).
  Milestones, phase boundaries, decisions, refactor notes — anything specific to how this plugin is built.
  Target audience: you, future-you, and GSD planning agents.

### Rationale

The marketplace copy is what users see on `claude plugin list` and in README tables. Phase numbers, milestone labels, and internal jargon leak context the user doesn't need and make releases look noisy. The two-file split keeps the user-facing log lean without losing the dev-internal history that makes the pipeline auditable.

### How Stage 3 primes you

The [Stage 3: Changelog Summary Prompt](#stage-3-changelog-summary-prompt) release-summary prompt names the taboos explicitly — phase numbers, milestone labels, internal jargon. This is the early-catch layer. Rewriting a leaked entry after rsync is more expensive than writing cleanly the first time. The same captured summary is reused by Stage 8a2 as the annotated tag's message, so any leaked jargon compounds into the tag history too.

### Rsync enforcement

[Stage 6: Package exclude list](#stage-6-package) contains `--exclude='CHANGELOG-internal.md'`. If the file doesn't exist, nothing is excluded — the rule is unconditional and safe. No `--delete` flag is used, so a pre-existing stale copy at a marketplace destination is NOT auto-removed; the manual remediation is a one-off `git rm` in the marketplace repo. No plugin currently has such a stale copy (verified for marketplace-manager; assumed for the rest until Phase 19 back-fill surfaces otherwise).

### Scaffolding and back-fill (Phase 19)

Phase 19 extends `/new-plugin` to scaffold both files with stubs (CLOG-05), and back-fills marketplace-manager's own `CHANGELOG-internal.md` from git history (CLOG-06). Phase 18 establishes the convention and the rsync enforcement only; it does NOT touch any existing plugin's on-disk CHANGELOGs. If a skill or plugin does not yet have `CHANGELOG-internal.md`, the pipeline runs clean — the rsync exclude is a no-op when there's nothing to exclude.

### Tagging note

[Sub-step 8a2](#stage-8-git-commit-and-push) creates an annotated source-repo tag `v<new-version>` on every publish, with the captured Stage 3 summary as the tag message. These publish tags are the new ground truth for Stage 2a's `git describe` baseline. The older milestone tags (`v1.0`, `v1.1`, `v1.2`) remain as archival markers in the repo history but no longer represent ship points. If you see two families of tags on a source repo, that's why.

### Example — lean entry (user-facing `CHANGELOG.md`)

    ## v0.3.2 — 2026-04-16

    Stage 2 now recommends a version bump from commit history.

### Example — rich entry (dev-internal `CHANGELOG-internal.md`)

    ## v0.3.2 — 2026-04-16 — Phase 18 complete

    - Added Stage 2a bump recommendation (BUMP-01..BUMP-06).
    - Stage 8a2 creates annotated source-repo tag.
    - CHANGELOG two-file split: user-facing vs dev-internal.
    - See .planning/phases/18-.../18-02-SUMMARY.md for verification trace.

---

## Stage 6: Package

**EXECUTE behaviour:**

### For skills (PUB-04) — ZIP with SKILL.md at root

**Critical:** Always `cd` into `claude-code-skill/` before running zip. If you run zip from outside that directory, the ZIP will contain `claude-code-skill/SKILL.md` instead of `SKILL.md` at root — Claude Desktop will fail to load the skill.

```bash
# Step 1: Rebuild the .skill ZIP
mkdir -p /Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/<name>/claude-desktop-skill/
cd /Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/<name>/claude-code-skill && zip -r ../claude-desktop-skill/<name>.skill . -x "evals/*" "*.pyc" "__pycache__/*" ".DS_Store"

# Step 2: Create marketplace destination and copy .skill file
mkdir -p /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/
cp /Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/<name>/claude-desktop-skill/<name>.skill /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill

# Step 3: Verify — SKILL.md must appear at root, NOT nested under a folder
unzip -l /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill
```

If the verify step shows `SKILL.md` nested under a directory prefix (e.g., `claude-code-skill/SKILL.md`), the package is invalid — stop with a mid-pipeline failure (D-08).

Read the file size for the output line:

```bash
ls -lh /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill | awk '{print $5}'
```

After packaging, print: `✅ Packaged: <name>.skill (<size>)`

### For plugins (PUB-05) — rsync copy with exclusions

**Pre-rsync cleanup:** Detect and remove any .git directories in the marketplace destination before copying. This catches contamination from prior publishes. The rsync `--exclude='.git/'` flag below prevents new contamination.

```bash
MARKETPLACE_PLUGIN="/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/plugins/<name>"

if [ -d "$MARKETPLACE_PLUGIN" ]; then
  CONTAMINATED=$(find "$MARKETPLACE_PLUGIN" -name ".git" -type d 2>/dev/null)
  if [ -n "$CONTAMINATED" ]; then
    echo "Cleaning .git contamination from marketplace copy..."
    rm -rf "$MARKETPLACE_PLUGIN/.git"
  fi
fi
```

Then proceed with the existing rsync command (unchanged).

```bash
rsync -a \
  --exclude='.git/' \
  --exclude='.planning/' \
  --exclude='.claude/' \
  --exclude='.vscode/' \
  --exclude='.idea/' \
  --exclude='.vs/' \
  --exclude='documentation/' \
  --exclude='CHANGELOG-internal.md' \
  --exclude='.DS_Store' \
  /Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/<name>/ \
  /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/plugins/<name>/
```

After copying, print: `✅ Packaged: <name> (rsync copy)`

**DRY-RUN behaviour:**

Do NOT run `zip`, `cp`, `mkdir -p`, `unzip -l`, `find`, `rm -rf`, or `rsync`. Do NOT create any temporary directories. Do NOT touch the marketplace destination.

Print exactly one line. The line shape depends on artifact type:

**For skills:**
```
[DRY RUN] ✅ Stage 6: Package — would zip claude-code-skill/ into <name>.skill and copy to <marketplace>/skills/<name>/<name>.skill
```

**For plugins:**
```
[DRY RUN] ✅ Stage 6: Package — would rsync <plugins-dir>/<name>/ to <marketplace>/plugins/<name>/ (excluding .git/, .planning/, .claude/, .vscode/, .idea/, .vs/, documentation/, CHANGELOG-internal.md, .DS_Store) after cleaning any .git contamination in destination
```

Use the actual resolved paths from Ecosystem Paths and Type Detection — substitute `<name>`, `<marketplace>`, and `<plugins-dir>` with their real values.

Return success. Continue to Stage 7.

---

## Stage 7: Copy to Marketplace and Update Index

**EXECUTE behaviour:**

### Sub-step A: Update marketplace.json (plugins only — PUB-06)

**Skills NEVER get marketplace.json entries.** Only plugins are registered in `marketplace.json`.

For plugins, use the read-parse-modify-rewrite pattern:

```python
import json

marketplace_json_path = '/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/.claude-plugin/marketplace.json'
plugin_json_path = '/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/<name>/.claude-plugin/plugin.json'
name = '<plugin-name>'
new_version = '<new-version>'

# Read description from plugin.json (already updated in Stage 4)
with open(plugin_json_path) as f:
    plugin_data = json.load(f)
new_description = plugin_data.get('description', '')

data = json.load(open(marketplace_json_path))

# New vs repeat detection
existing = next((p for p in data['plugins'] if p['name'] == name), None)
if existing:
    # Repeat publish: update fields in place
    existing['version'] = new_version
    existing['description'] = new_description
else:
    # New publish: append entry to plugins array
    data['plugins'].append({
        'name': name,
        'source': f'./plugins/{name}',
        'description': new_description,
        'version': new_version
    })

with open(marketplace_json_path, 'w') as f:
    json.dump(data, f, indent=2)
    f.write('\n')
```

After updating, print: `✅ Copied to marketplace`

For skills: skip marketplace.json update entirely, but still print: `✅ Copied to marketplace`

### Sub-step A2: Update plugin README version line (plugins only)

**Skills do not have conventional README files.** Skip this sub-step entirely for skills — no file checks, no output.

For plugins, update the version line in the plugin's README:

**If source README does not exist** (PREAD-03 — scaffold generation):

```python
import json, os

source_readme = '<source-dir>/README.md'
marketplace_readme = '<marketplace>/plugins/<name>/README.md'
plugin_json_path = '<source-dir>/.claude-plugin/plugin.json'
new_version = '<new-version>'  # from Stage 2

with open(plugin_json_path) as f:
    pdata = json.load(f)

scaffold = f"""# {pdata.get('name', '<name>')}

{pdata.get('description', '')}

## Installation

\```bash
claude plugin add {pdata.get('name', '<name>')}
\```

## Version

Current version: `{new_version}` — see [CHANGELOG.md](CHANGELOG.md) for release history.

## License

Personal use. Part of Mikey East's Claude ecosystem.
"""

with open(source_readme, 'w') as f:
    f.write(scaffold)
with open(marketplace_readme, 'w') as f:
    f.write(scaffold)
```

Print: `✅ README scaffolded for <name> v<new-version>`

**If source README exists** (PREAD-01, PREAD-02 — version line update):

```python
import re

source_readme = '<source-dir>/README.md'
marketplace_readme = '<marketplace>/plugins/<name>/README.md'
new_version = '<new-version>'  # from Stage 2

with open(source_readme) as f:
    existing = f.read()

new_content = re.sub(
    r'(Current version:\s*`)[^`]+(`)',
    r'\g<1>' + new_version + r'\g<2>',
    existing
)

if new_content == existing:
    if not re.search(r'Current version:\s*`[^`]+`', existing):
        # PIPE-02: Warn and continue — do NOT hard-stop
        print(f"⚠️  README: version line pattern not found in {source_readme} — README not updated")
    # else: version unchanged (same version re-published) — D-08: no output, no write
else:
    old_match = re.search(r'Current version:\s*`([^`]+)`', existing)
    old_version = old_match.group(1) if old_match else '?'

    with open(source_readme, 'w') as f:
        f.write(new_content)
    with open(marketplace_readme, 'w') as f:
        f.write(new_content)

    # D-07: Inline summary matching Stage 4 pattern
    print(f"✅ README updated: v{old_version} → v{new_version}")
```

Use `<source-dir>`, `<marketplace>`, `<name>`, and `<new-version>` as placeholders matching the existing Stage 7a style — the executor resolves them from Ecosystem Paths and Type Detection at runtime.

### Sub-step B: Regenerate README.md (PUB-07)

Regenerate the full marketplace README.md after every publish — it reflects the current state of ALL published skills and plugins, not just the one being published.

**Step 1: Scan marketplace for all published skills**

```bash
ls /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/ 2>/dev/null
```

For each skill directory found, read version and description from the `.skill` ZIP:

```bash
# Read version from .skill ZIP
unzip -p /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill "SKILL.md" 2>/dev/null | python3 -c "
import re, sys
content = sys.stdin.read()
m = re.search(r'^---\n(.*?)^---', content, re.MULTILINE | re.DOTALL)
if m:
    vm = re.search(r'^version:\s*(.+)$', m.group(1), re.MULTILINE)
    print(vm.group(1).strip() if vm else 'MISSING')
else:
    print('MISSING')
"

# Read description from .skill ZIP (first sentence recommended for table cell brevity)
unzip -p /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill "SKILL.md" 2>/dev/null | python3 -c "
import re, sys
content = sys.stdin.read()
m = re.search(r'^---\n(.*?)^---', content, re.MULTILINE | re.DOTALL)
if m:
    dm = re.search(r'^description:\s*(.+)$', m.group(1), re.MULTILINE)
    desc = dm.group(1).strip().strip('\"') if dm else '—'
    # Use first sentence for table brevity
    first_sentence = desc.split('.')[0].strip()
    print(first_sentence if first_sentence else '—')
else:
    print('—')
"
```

**Step 2: Scan marketplace for all published plugins**

```bash
ls /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/plugins/ 2>/dev/null
```

For each plugin directory found, read version and description from `.claude-plugin/plugin.json`:

```python
import json
d = json.load(open('/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/plugins/<name>/.claude-plugin/plugin.json'))
version = d.get('version', '—')
description = d.get('description', '—')
```

**Step 3: Assemble README.md rows**

- If version reads as `MISSING`, show `—` in the table cell (Pitfall 4 safeguard)
- Sort rows alphabetically by name within each table
- Plugin rows include a fourth column: the install command `claude plugin add <name>` where `<name>` is the plugin name from `plugin.json`

**Step 4: Write README.md (skip-if-unchanged guard)**

Generate the README content using the D-07 format contract, then compare against the existing file before writing:

```markdown
# Mikey Skills Marketplace

Skills and plugins for Claude Code and Claude Desktop.

## Skills

| Skill | Version | Description |
|-------|---------|-------------|
| <name> | <version> | <description> |

## Plugins

| Plugin | Version | Description | Install |
|--------|---------|-------------|---------|
| <name> | <version> | <description> | `claude plugin add <name>` |

---
*Auto-generated by marketplace-manager*
```

Assemble the full README string (`new_content`) from the D-07 template above, populated with the rows from Step 3.

If there are no published skills yet, omit the Skills table rows (keep the header). If there are no published plugins yet, omit the Plugins table rows (keep the header).

Then apply the skip-if-unchanged guard before writing:

```python
readme_path = '/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/README.md'

# Read existing README for comparison
try:
    with open(readme_path) as f:
        existing_content = f.read()
except FileNotFoundError:
    existing_content = ''

if new_content == existing_content:
    # MREAD-03: skip-if-unchanged with visible output (D-07, D-08, D-09)
    print('(skip) Stage 7b: README.md unchanged — skipped')
else:
    with open(readme_path, 'w') as f:
        f.write(new_content)
    print('✅ README.md index regenerated')
```

**DRY-RUN behaviour:**

Do NOT open `marketplace.json` for writing. Do NOT scan the marketplace skills or plugins directory. Do NOT regenerate `README.md`. Do NOT open any README for reading or writing. Do NOT run any JSON write commands.

Print exactly two lines for skills, three lines for plugins.

**Line 1 — marketplace.json (plugins only; skills print the skip variant):**

For plugins:
```
[DRY RUN] ✅ Stage 7a: marketplace.json — would update entry "<name>" → version <new-version>
```

For skills:
```
[DRY RUN] ✅ Stage 7a: marketplace.json — would skip marketplace.json (skills do not register)
```

**Line 2 — plugin README version (plugins only; skills skip entirely):**

For plugins:
```
[DRY RUN] ✅ Stage 7a2: README — would update version to <new-version> in <source-readme-path> and <marketplace-readme-path>
```

For skills: do NOT print any 7a2 line.

**Line 3 — README.md index (both types; Line 2 for skills):**
```
[DRY RUN] ✅ Stage 7b: README.md — would regenerate marketplace index with <name> v<new-version>
```

Return success. Continue to Stage 8.

---

### README Change Summary (between Stage 7 and Stage 8)

**EXECUTE behaviour only** — this block does NOT run in DRY-RUN mode.

After Stage 7b completes (whether it wrote the file or skipped), and BEFORE Stage 8 begins, display a structured change summary if the README was updated (per D-01, D-02, D-03).

If Stage 7b printed the skip line (content unchanged), do NOT print any summary — the skip line is the signal (per D-09).

If Stage 7b wrote the file (content changed), compare old rows vs new rows to generate the summary:

```python
import re

def extract_rows(content):
    """Extract {name: version} from a generated README table.
    Handles both old flat 'Available' table and new split tables."""
    rows = {}
    for line in content.splitlines():
        m = re.match(r'\|\s*([^|]+?)\s*\|\s*([^|]+?)\s*\|', line)
        if m:
            name = m.group(1).strip()
            version = m.group(2).strip()
            # Skip header and separator rows
            if name.lower() not in ('name', 'skill', 'plugin', '---') and \
               version.lower() not in ('version', '---') and \
               not name.startswith('-'):
                rows[name] = version
    return rows

old_rows = extract_rows(existing_content)
new_rows = extract_rows(new_content)

all_names = sorted(set(old_rows) | set(new_rows))
changes = []
for name in all_names:
    if name not in old_rows:
        changes.append(f"  + Added: {name} v{new_rows[name]}")
    elif name not in new_rows:
        changes.append(f"  - Removed: {name}")
    elif old_rows[name] != new_rows[name]:
        changes.append(f"  ~ Updated: {name} v{old_rows[name]} -> v{new_rows[name]}")

if changes:
    print("")
    print("README changes:")
    for line in changes:
        print(line)
    print("")
```

Note: `existing_content` and `new_content` are the same variables from Step 4's skip guard — they are already in scope. The `extract_rows` function is permissive: it handles both the old flat "Available" table format (first run after Phase 15) and the new split Skills/Plugins table format (all subsequent runs). On the first run, entries will show as "Updated" if their version matches but the table structure changed — this is acceptable.

The diff summary uses `+`, `-`, `~` prefixes for Added, Removed, Updated respectively.

---

## Stage 8: Git Commit and Push

**Requirements:** PUB-08, PUB-09

**EXECUTE behaviour:**

### Sub-step 8a: Marketplace commit (after gate)

```bash
MARKETPLACE=/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace
# Stage all changes FIRST (required before STAGED_COUNT and gate prompt)
git -C "$MARKETPLACE" add .
STAGED_COUNT=$(git -C "$MARKETPLACE" diff --cached --name-only | wc -l | tr -d ' ')
echo ""
echo "Ready to push:"
echo "  Commit: publish: <name> v<new-version> — <summary>"
echo "  Files staged: $STAGED_COUNT"
echo ""
echo "Push to GitHub? (y/N)"
read -r GATE_ANSWER
case "$GATE_ANSWER" in
  [yY]) ;;  # proceed
  *)
    # Gate abort — show D-13 gate-abort format, stop. Local writes preserved.
    exit ;;
esac
git -C "$MARKETPLACE" commit -m "publish: <name> v<new-version> — <summary>"
```

### Sub-step 8a2: Git Tag (source repo)

After the marketplace commit (8a) and BEFORE the push (8b), create an annotated tag on the source repo (NOT the marketplace repo) so that the next publish has a fresh baseline for Stage 2a's `git describe` (closes the BUMP-01 feedback loop). Runs UNCONDITIONALLY except when the source repo has no `.git/` directory (D-07 skip).

**Rationale notes:**
- Why source repo (not marketplace): Stage 2a's `git describe` reads the source repo (D-02); marketplace tag space would collide across plugins; source is per-plugin.
- Why no tag push: out of scope per CONTEXT (deferred). Stage 8b push target is marketplace repo, unchanged.
- Why no post-create assertion: per RESEARCH Q10, `git tag -a` is atomic; the Pitfall-4 existence guard below covers the retry case.
- Why the tag points at pre-bump HEAD: Stage 4/5 edits are uncommitted at this point; the tag is a publish-time anchor, not a commit-correctness claim. Future phases introducing source-repo commits can move the tag forward.

```bash
SOURCE_DIR="<source-dir>"   # container_dir, same as Stage 5
NEW_VERSION="<new-version>" # from Stage 2b
# SUMMARY was captured in Stage 3

# D-07: no source git repo -> warn-and-continue (matches PIPE-02)
if ! git -C "$SOURCE_DIR" rev-parse --show-toplevel >/dev/null 2>&1; then
  echo "⚠️  Tag skipped — <name> has no source git repo"
else
  # Pitfall 4: retry-publish guard; do NOT force-move an existing tag
  if git -C "$SOURCE_DIR" tag -l "v${NEW_VERSION}" | grep -q "v${NEW_VERSION}"; then
    echo "⚠️  Tag v${NEW_VERSION} already exists on source repo — leaving in place"
  else
    git -C "$SOURCE_DIR" tag -a "v${NEW_VERSION}" -m "$SUMMARY"
    echo "✅ Tagged source: v${NEW_VERSION}"
  fi
fi
```

**Threat note (T-18-04):** `SUMMARY` is passed as a single `-m "$SUMMARY"` argv value — git does NOT re-interpret the value as shell. Any metacharacters (`;`, `$`, backticks) land in the tag message verbatim.

### Sub-step 8b: Push

```bash
git -C "$MARKETPLACE" push origin main
PUSH_EXIT=$?
```

If `PUSH_EXIT` is 0 (success):
- Print: `✅ Committed and pushed: publish: <name> v<new-version> — <summary>`
- Print final success line: `✔ Published <name> v<new-version>`
- Print verification nudge: `Verify the update is live: https://github.com/8bit-ginge/mikey-skills-marketplace`
- Pipeline complete.

If `PUSH_EXIT` is not 0 (failure): follow the two-tier recovery protocol defined in Scenario C (D-10). Summary:
- Tier 1: print `Push failed — attempting auto-recovery`, run `git -C "$MARKETPLACE" pull --rebase origin main`, capture `REBASE_EXIT`
  - If `REBASE_EXIT` is 0: print `Rebase succeeded — retrying push`, retry push, capture `RETRY_EXIT`
    - If `RETRY_EXIT` is 0: print same success lines as above (checkmark + Published + verification nudge)
    - If `RETRY_EXIT` is not 0: fall through to Tier 2 (Scenario C Case A)
  - If `REBASE_EXIT` is not 0: run `git -C "$MARKETPLACE" rebase --abort` IMMEDIATELY, print `Rebase aborted — marketplace repo restored to clean state`, fall through to Tier 2 (Scenario C Case B)

**DRY-RUN behaviour:**

Do NOT run `git add`, `git commit`, or `git push`. Do NOT run `git branch`, `git remote`, `git status`, `git ls-remote`, `git rev-list`, or ANY other write-path git subcommand. Do NOT resolve the current branch name or remote URL at runtime — use the static string `origin/main` and the marketplace path from Ecosystem Paths.

**Exception for Stage 8a2 (per D-19):** the read-only detection `git -C "$SOURCE_DIR" rev-parse --show-toplevel >/dev/null 2>&1` AND the read-only guard `git -C "$SOURCE_DIR" tag -l "v${NEW_VERSION}"` ARE run in dry-run so the printed verb phrase is faithful (normal / D-07 / Pitfall-4). These reads never write — they only select which of the three 8a2 lines below to print. Same precedent as Stage 0c's `git ls-remote` and Stage 2a's `git describe` under D-19.

Rationale: D-11 prohibits git WRITES in dry-run Stage 8. Stage 8a2's read-only detection is allowed because a misleading `would create annotated tag` line for a skill that has no `.git` directory violates D-12 faithfulness. The marketplace remote is single-branch by project convention (see Ecosystem Paths), so no runtime resolution is necessary.

Print exactly five lines. For the Stage 8a2 line, select ONE of three variants based on the real read-only detection above:

**Normal path** (source repo exists; tag not yet present):

```
[DRY RUN] ✅ Stage 8 Gate — would prompt for pre-push confirmation (y/N)
[DRY RUN] ✅ Stage 8a: Git — would commit "publish: <name> v<new-version> — <summary>" in <marketplace>
[DRY RUN] ✅ Stage 8a2: Git tag — would create annotated tag v<new-version> on <source-dir>
[DRY RUN] ✅ Stage 8b: Git — would push to origin/main
[DRY RUN] ✅ Would prompt to verify update on GitHub
```

**D-07 path** (no source git repo) — replace the Stage 8a2 line with:

```
[DRY RUN] ⚠️  Stage 8a2: Git tag — would skip (no source git repo for <name>)
```

**Pitfall-4 path** (tag already exists on source repo) — replace the Stage 8a2 line with:

```
[DRY RUN] ⚠️  Stage 8a2: Git tag — would skip (tag v<new-version> already exists on <source-dir>)
```

Where:
- `<name>` is the artifact name
- `<new-version>` is the value chosen at Stage 2
- `<summary>` is the changelog summary captured at Stage 3
- `<marketplace>` is the marketplace path from Ecosystem Paths (do NOT resolve via git)

Then print the D-12 final success line:

```
✔ [DRY RUN] Simulated <name> v<new-version> — no files written
```

Then print the D-12 summary table (see Output Format Contracts → Dry-Run Output (D-12) for the schema). The table is the LAST output of the command — nothing follows it. Return success. The pipeline is complete.

On PREFLIGHT or mid-pipeline hard-stop failure: the summary table is STILL printed per D-12, with the failed stage row marked `FAIL` and subsequent rows marked `SKIP`. The `[DRY RUN] ❌` failure block appears immediately before the table.

---

## Output Format Contracts

### Validation Gate Failure (D-03)

Display the full validation report (same as `/marketplace-manager:validate` output):

```
## validate: <name>

❌ <check category>: <check name> — <failure reason>
...

X check(s) failed.
Fix the issues above, then run `/marketplace-manager:publish <name>` again.
```

### Successful Publish (D-02)

Each pipeline stage prints its tick line IMMEDIATELY after completing — not all at the end. The output builds progressively as the user watches.

```
## publish: <name>

✅ Validation passed
✅ Version bumped: <old-version> → <new-version>
✅ CHANGELOG.md updated
✅ Packaged: <name>.<ext> (<size>)
✅ Copied to marketplace
✅ README.md index regenerated
✅ Committed and pushed: publish: <name> v<new-version> — <summary>

✔ Published <name> v<new-version>

Verify the update is live: https://github.com/8bit-ginge/mikey-skills-marketplace
```

The `✔` on the final line uses the heavy check mark (U+2714) to distinguish the completion summary from the per-stage tick marks (U+2705).

For skills, the packaged line shows: `✅ Packaged: <name>.skill (<size>)`
For plugins, the packaged line shows: `✅ Packaged: <name> (rsync copy)`

### PREFLIGHT Output (D-11)

PREFLIGHT prints one line per check immediately after each check completes. The block is bounded by `--- Preflight ---` and `--- Pipeline ---` delimiter lines.

**All checks pass:**

```
--- Preflight ---
✅ Semver: <proposed-version> is valid
✅ Version: <proposed-version> > current <current-version>
✅ Remote: no commits ahead of local
✅ Manifest: <manifest-type> found and valid
--- Pipeline ---
```

**On warning (check 0b or 0c):**

```
⚠️  Version: <proposed> is not greater than <current> — Continue anyway? (y/N)
```

**On hard stop (check 0a or 0d):**

```
❌ <check name>: <failure description>

**What happened:** <plain-language explanation>

**Your changes are safe** — no files have been written. Nothing was lost.

**To fix:** <specific recovery action>
```

### Dry-Run Output (D-12)

Triggered when the user invokes `/marketplace-manager:publish <name> --dry-run`. All output lines in this mode are prefixed with `[DRY RUN]`. No files are created, modified, or pushed. This contract applies ONLY in dry-run mode; execute mode keeps its D-02 streaming format unchanged and prints NO summary table.

**Prefix rule:** Every line of dry-run output — including stage tick lines, PREFLIGHT check lines, warning prompts (0b, 0c), hard-stop failure blocks, and the final ✔ line — starts with `[DRY RUN] ` (note the trailing space). The only exceptions are:

- The `--- Preflight ---` and `--- Pipeline ---` delimiter lines (structural — no prefix)
- Individual rows of the summary table (bounded by its own markdown pipe syntax — no per-row prefix)

**Per-stage line shape:**

```
[DRY RUN] ✅ Stage <N>: <stage name> — would <verb phrase>
```

The verb phrase names the concrete target: file path, command summary, or remote destination. Use the actual values resolved during Stages 1–3 and PREFLIGHT (proposed version from Stage 2, summary from Stage 3, type-detected container paths from Type Detection), not placeholders.

**Glyph vocabulary** (matches D-02):

- `✅` — this stage would succeed if run for real
- `⚠️` — warning, user must confirm (PREFLIGHT checks 0b, 0c)
- `❌` — this stage would fail; pipeline stops here
- `✔` (final line only) — simulation complete

**Per-stage verb phrases** (authoritative — stage DRY-RUN behaviour blocks MUST transcribe these exactly; these strings are copy-paste targets — do not paraphrase):

| Stage | Verb phrase template |
|-------|----------------------|
| 1 | `would run all validation checks (<pass-count> passed)` |
| 2a | `would compute recommendation (<type> — <rationale>)` |
| 2b | `would prompt for bump type (selected: <type> → <new-version>)` |
| 0 | `would run 4 checks (<status>)` |
| 3 | `would prompt for user-facing summary (captured)` |
| 4 | `would write version <new-version> to <manifest-file>` |
| 5 | `would prepend v<new-version> entry to <container>/CHANGELOG.md` |
| 6 (plugin) | `would rsync <plugins-dir>/<name>/ to <marketplace>/plugins/<name>/ (excluding .git/, .planning/, .claude/, .vscode/, .idea/, .vs/, documentation/, CHANGELOG-internal.md, .DS_Store) after cleaning any .git contamination in destination` |
| 6 (skill) | `would zip claude-code-skill/ into <name>.skill and copy to <marketplace>/skills/<name>/<name>.skill` |
| 7a | `would update marketplace.json entry "<name>" → version <new-version>` (plugins only) |
| 7a2 | `would update version to <new-version> in <source-readme-path> and <marketplace-readme-path>` (plugins only; skills skip — no 7a2 line printed) |
| 7b | `would regenerate marketplace index with <name> v<new-version>` |
| 8a | `would commit "publish: <name> v<new-version> — <summary>" in <marketplace>` |
| 8a2 | `would create annotated tag v<new-version> on <source-dir>` |
| 8b | `would push to origin/main` |

For skills at Stage 7a, the line is `would skip marketplace.json (skills do not register)`.

**Final success line:**

```
✔ [DRY RUN] Simulated <name> v<proposed-version> — no files written
```

The `✔` (heavy check mark, U+2714) leads the line — `[DRY RUN]` follows the glyph because ✔ is the completion sigil that must be first-scan visible.

**Summary table** — printed LAST, after the ✔ line, using GitHub-flavoured markdown pipes:

```
| Stage | Name                        | Status | What would happen                                     |
|-------|-----------------------------|--------|-------------------------------------------------------|
| 1     | Validation Gate             | PASS   | would run all validation checks (<pass-count> passed) |
| 2a    | Bump Recommendation         | PASS   | would compute recommendation (<type> — <rationale>)   |
| 2b    | Version Bump Prompt         | PASS   | would prompt for bump type (selected: <type> → <new-version>) |
| 0     | PREFLIGHT                   | PASS   | would run 4 checks (<status>)                         |
| 3     | Changelog Summary Prompt    | PASS   | would prompt for user-facing summary (captured)       |
| 4     | Write Version Bump          | PASS   | would write version <new-version> to <manifest-file>  |
| 5     | Write CHANGELOG Entry       | PASS   | would prepend v<new-version> entry to <container>/CHANGELOG.md |
| 6     | Package                     | PASS   | would rsync <plugins-dir>/<name>/ to <marketplace>/plugins/<name>/ (excluding .git/, .planning/, .claude/, .vscode/, .idea/, .vs/, documentation/, CHANGELOG-internal.md, .DS_Store) after cleaning .git contamination / would zip claude-code-skill/ into <name>.skill and copy to <marketplace>/skills/<name>/<name>.skill |
| 7     | Copy to Marketplace + Index | PASS   | would update marketplace.json, update README version, and regenerate index with <name> v<new-version> |
| 8     | Git Commit and Push         | PASS   | would commit marketplace, tag source repo v<new-version>, push to origin/main |
```

**Row order:** Execution order, matching the Pipeline Overview table (1, 2, 0, 3, 4, 5, 6, 7, 8). Do NOT sort numerically — Stage 0 (PREFLIGHT) appears in the third row because it runs between Stage 2 and Stage 3.

**Status values:**

- `PASS` — stage would succeed if run for real
- `WARN` — stage raised a warning (PREFLIGHT 0b regression, 0c remote-ahead) and the user chose to continue
- `FAIL` — stage would fail (hard stop); pipeline stopped at this stage
- `SKIP` — stage was not reached because an earlier stage failed or the user declined a warning

**On hard-stop failure:** The summary table is STILL printed. The failed stage's row shows `FAIL`, and all subsequent stages show `SKIP` with the "What would happen" cell reading `— skipped (earlier stage failed)` or `— skipped (user declined warning)`. The per-stage streaming lines and the `[DRY RUN] ❌` failure block appear above the table as normal. Rationale: the summary table is the primary artifact of dry-run mode (D-10), and a user who hits a preflight hard stop still needs to see what the later stages would have been.

**Warning prompt prefixing:** PREFLIGHT warning prompts in dry-run mode carry the prefix INSIDE the line. Example:

```
[DRY RUN] ⚠️  Version: 1.2.0 is not greater than 1.3.0 — Continue anyway? (y/N)
```

**Hard-stop failure block prefixing:** Only the `❌` header line carries the `[DRY RUN] ` prefix. The three-section body ("What happened" / "Your changes are safe" / "To fix") inherits the prefix's scope and is NOT line-by-line prefixed. This matches the treatment of the same block in D-11.

**Execute mode exclusion:** The summary table is dry-run exclusive. Execute mode keeps its existing D-02 streaming-ticks-plus-final-✔ format unchanged — no table is printed in execute mode under any circumstance.
### Pre-Push Gate Output (D-13)

**Gate prompt (fires between Stage 7 completion and git commit):**

```
Ready to push:
  Commit: publish: <name> v<new-version> — <summary>
  Files staged: <count>

Push to GitHub? (y/N)
```

**Gate abort (user answers N or presses Enter):**

```
❌ Publish aborted

**What happened:** You chose not to push. The pipeline stopped before any git operations beyond staging.

**Your changes are safe** — version bump, changelog, and marketplace files are saved locally. Nothing was lost.

**To push later:** Run these commands in your terminal:
  git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace commit -m "publish: <name> v<new-version> — <summary>"
  git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace push origin main
```

Note: `git add .` has already run before the gate prompt, so the abort recovery only needs commit + push.

**Verification nudge (per D-13 — success path only, never on failure or abort):**

```
Verify the update is live: https://github.com/8bit-ginge/mikey-skills-marketplace
```

Informational only — no y/N, no waiting. Pipeline ends after printing it.

---

## Failure Handling

### Scenario A: Validation Gate Failure (D-09)

Triggered when Stage 1 finds any failed checks.

Display the full validation report in standard D-03 format (same as `/marketplace-manager:validate` output), then append:

```
Fix the issues above, then run `/marketplace-manager:publish <name>` again.
```

No further pipeline stages execute. No writes have occurred.

---

### Scenario B: Mid-Pipeline Failure (D-08)

Triggered when any stage from Stage 4 onwards encounters an error (file write error, zip error, rsync error, etc.).

Output shows:
1. All completed stages with ✅ prefix (everything that succeeded before the failure)
2. The failed stage with ❌ prefix

Then three plain-language sections:

```
❌ <stage description> failed

**What happened:** <plain-language explanation of what went wrong>

**Your changes are safe** — everything written before this step is preserved on disk. Nothing was lost.

**To fix:** <specific recovery command or action>
```

**Example for zip verification failure:**

```
✅ Validation passed
✅ Version bumped: 2 → 3
✅ CHANGELOG.md updated
❌ Package verification failed

**What happened:** The ZIP file was created but SKILL.md is nested under a folder instead of at the root. Claude Desktop will not be able to load this skill.

**Your changes are safe** — the version bump and changelog entry are saved locally. Nothing was lost.

**To fix:** Run this command in your terminal:
  cd /Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/<name>/claude-code-skill && zip -r ../claude-desktop-skill/<name>.skill . -x "evals/*" "*.pyc" "__pycache__/*" ".DS_Store"
```

---

### Scenario C: Git Push Failure (D-10)

Triggered when `git push` returns a non-zero exit code.

**Tier 1 — Auto-recovery:**

Attempt exactly one pull-rebase cycle:
1. Print: `Push failed — attempting auto-recovery`
2. Print: `Pulling latest from origin/main...`
3. Run: `git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace pull --rebase origin main`
4. If rebase succeeds (exit code 0):
   - Print: `Rebase succeeded — retrying push`
   - Run: `git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace push origin main`
   - If retry push succeeds: continue to success path (✅ + ✔ + verification nudge)
   - If retry push fails: fall through to Tier 2
5. If rebase fails (exit code non-zero):
   - Run IMMEDIATELY: `git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace rebase --abort`
   - Print: `Rebase aborted — marketplace repo restored to clean state`
   - Fall through to Tier 2 with rebase-failed wording

**Tier 2 — Manual recovery (preserved D-10 format):**

Show all completed stages with ✅, then show the appropriate message:

**Case A — Retry push failed (rebase succeeded but push still fails):**
```
❌ Git push failed

**What happened:** The push failed even after an automatic rebase. The remote may have other conflicts or access restrictions.

**Your changes are safe** — all local writes are preserved. The commit exists locally. Nothing was lost.

**To fix:** Run this command in your terminal:
  git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace push origin main
```

**Case B — Rebase conflict (rebase itself failed):**
```
❌ Auto-recovery failed

**What happened:** The push failed and the automatic rebase hit a conflict. The rebase has been aborted — the marketplace repo is in a clean state.

**Your changes are safe** — all local writes are preserved. Nothing was lost.

**To fix:** Manually pull and resolve conflicts, then push:
  git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace pull --rebase origin main
  # resolve any conflicts
  git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace push origin main
```

Do NOT attempt rollback of local changes. Do NOT attempt more than one auto-recovery cycle.

---

### Scenario D: PREFLIGHT Failure

Triggered when check 0a (semver validity) or check 0d (manifest presence) fails.

Display all completed preflight check lines (✅ or ⚠️) up to the failure point, then the ❌ line with the three-section failure format (What happened / Your changes are safe / To fix).

No pipeline stages after PREFLIGHT execute. No writes have occurred.

---

### Scenario E: Dry-Run Termination (D-12 summary table on every exit path)

**Applies when:** `MODE=dry-run` and the pipeline is about to exit for ANY reason — successful completion of Stage 8, a hard-stop failure at any earlier stage (PREFLIGHT 0a/0d semver or manifest-mismatch, Stage 1 validation, or any write-stage simulation failure), or a warning-prompt abort (PREFLIGHT 0b/0c where the user answers N).

**Rule (cross-cutting — applies to every stage, not just Stage 8):** In dry-run mode, the very last output before exit MUST be the D-12 summary table. No exceptions. The table shows one row per stage (0, 1, 2, 3, 4, 5, 6, 7, 8) with status `PASS`, `WARN`, `FAIL`, or `SKIP`:

- `PASS` — stage ran to completion (read-only stage succeeded, or write stage printed its `[DRY RUN] ✅` line)
- `WARN` — stage surfaced a warning but continued (e.g. PREFLIGHT 0b version-regression warn-continue)
- `FAIL` — stage hard-stopped (e.g. PREFLIGHT 0a semver, Stage 1 validation gate, or user answered N to a warning prompt)
- `SKIP` — stage was never reached because an earlier stage failed

**Termination paths:**

1. **Successful completion** — Stage 8 prints the `✔ [DRY RUN] Simulated ...` line, THEN the summary table (see Stage 8 DRY-RUN behaviour block).
2. **Hard-stop failure** — the stage that failed prints its `[DRY RUN] ❌` failure block (using the standard D-08/D-09/D-10 three-section format, prefixed with `[DRY RUN]` per the Pipeline Overview prefix rule), THEN the summary table with the failed stage row marked `FAIL` and all later rows marked `SKIP`. The table is printed BEFORE the process exits non-zero.
3. **Warning-prompt abort** — PREFLIGHT check 0b or 0c prompts `Continue anyway? (y/N)`. If the user answers N, the stage prints its `[DRY RUN]` abort line, THEN the summary table with the aborted stage row marked `WARN` and all later rows marked `SKIP`.

**Implementation note for executors:** Because the summary table must appear on failure paths as well as success, every hard-stop failure handler in read-only stages (1, 2, 0, 3) and every simulated failure in write stages (4-8) must, when `MODE=dry-run`, emit the table as its final action BEFORE exiting. The Stage 8 DRY-RUN behaviour block (where the success path prints the table) is NOT the only place the table lives — it is the success-path implementation of this cross-cutting rule.

**Schema:** See Output Format Contracts → D-12 Dry-Run Output Format for the column layout (Stage | Name | Status | What would happen) and an example rendering.

---

*This file is the complete pipeline logic reference. SKILL.md loads this file explicitly via `Read: ${CLAUDE_SKILL_DIR}/references/publish-rules.md` before executing any pipeline stage.*
