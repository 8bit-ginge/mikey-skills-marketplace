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
10. [Stage 6: Package](#stage-6-package)
11. [Stage 7: Copy to Marketplace and Update Index](#stage-7-copy-to-marketplace-and-update-index)
12. [Stage 8: Git Commit and Push](#stage-8-git-commit-and-push)
13. [Output Format Contracts](#output-format-contracts)
    - [Dry-Run Output (D-12)](#dry-run-output-d-12)
14. [Failure Handling](#failure-handling)

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

## Stage 2: Version Bump Prompt

**Requirement:** PUB-03 — present version bump options before any writes.

### Step 1: Read current version

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

### Step 2: Normalize version to semver and compute bump options

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

### Step 3: Present the prompt (D-04 format)

```
Current version: <version>

What kind of change is this?
1. Patch (bug fix) → <patch_ver>
2. Minor (new feature) → <minor_ver>
3. Major (breaking change) → <major_ver>
```

Wait for user response (1, 2, or 3). Store the selected new version as `<new-version>`.

**The selected version is always in semver format** (e.g., `2.0.1`), regardless of the original format. This aligns with the project convention that versions follow `X.X.X`.

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

Present exactly:

```
Describe the change in one sentence:
```

Wait for user response. Store as `<summary>`.

Do NOT combine this prompt with the version bump prompt. Two separate interactions — one thing at a time (D-03).

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

```python
changelog_path = '/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-<type>s/<name>/CHANGELOG.md'
new_version = '<new-version>'
today = '<YYYY-MM-DD>'
summary = '<summary>'

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
    flags=re.MULTILINE
)

if new_content == content:
    raise ValueError(
        'CHANGELOG.md does not contain a "# Changelog" header — '
        'entry was not inserted. Add the header and retry.'
    )

with open(changelog_path, 'w') as f:
    f.write(new_content)
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
  --exclude='documentation/' \
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
[DRY RUN] ✅ Stage 6: Package — would rsync <plugins-dir>/<name>/ to <marketplace>/plugins/<name>/ (excluding .git/, .planning/, .claude/, documentation/, .DS_Store) after cleaning any .git contamination in destination
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

**Step 4: Write README.md**

Generate and write the README using the D-07 format contract:

```markdown
# Mikey Skills Marketplace

Skills and plugins for Claude Code and Claude Desktop.

## Skills

| Skill | Version | Description |
|-------|---------|-------------|
| <name> | <version> | <description> |

## Plugins

| Plugin | Version | Description |
|--------|---------|-------------|
| <name> | <version> | <description> |

---
*Auto-generated by marketplace-manager*
```

Write to `/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/README.md`.

If there are no published skills yet, omit the Skills table rows (keep the header). If there are no published plugins yet, omit the Plugins table rows (keep the header).

After writing, print: `✅ README.md index regenerated`

**DRY-RUN behaviour:**

Do NOT open `marketplace.json` for writing. Do NOT scan the marketplace skills or plugins directory. Do NOT regenerate `README.md`. Do NOT run any JSON write commands.

Print exactly two lines.

**Line 1 — marketplace.json (plugins only; skills print the skip variant):**

For plugins:
```
[DRY RUN] ✅ Stage 7a: marketplace.json — would update entry "<name>" → version <new-version>
```

For skills:
```
[DRY RUN] ✅ Stage 7a: marketplace.json — would skip marketplace.json (skills do not register)
```

**Line 2 — README.md index (both types):**
```
[DRY RUN] ✅ Stage 7b: README.md — would regenerate marketplace index with <name> v<new-version>
```

Return success. Continue to Stage 8.

---

## Stage 8: Git Commit and Push

**Requirements:** PUB-08, PUB-09

**EXECUTE behaviour:**

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

Do NOT run `git add`, `git commit`, or `git push`. Do NOT run `git -C`, `git branch`, `git remote`, `git status`, `git rev-parse`, `git ls-remote`, `git rev-list`, or ANY other git subcommand. Do NOT resolve the current branch name or remote URL at runtime — use the static string `origin/main` and the marketplace path from Ecosystem Paths.

Rationale: D-11 prohibits ALL git invocations in dry-run Stage 8 to keep the simulation pure. The marketplace remote is single-branch by project convention (see Ecosystem Paths), so no runtime resolution is necessary.

Print exactly four lines:

```
[DRY RUN] ✅ Stage 8 Gate — would prompt for pre-push confirmation (y/N)
[DRY RUN] ✅ Stage 8a: Git — would commit "publish: <name> v<new-version> — <summary>" in <marketplace>
[DRY RUN] ✅ Stage 8b: Git — would push to origin/main
[DRY RUN] ✅ Would prompt to verify update on GitHub
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
| 2 | `would prompt for bump type (selected: <type> → <new-version>)` |
| 0 | `would run 4 checks (<status>)` |
| 3 | `would prompt for summary (captured)` |
| 4 | `would write version <new-version> to <manifest-file>` |
| 5 | `would prepend v<new-version> entry to <container>/CHANGELOG.md` |
| 6 (plugin) | `would rsync <plugins-dir>/<name>/ to <marketplace>/plugins/<name>/ (excluding .git/, .planning/, .claude/, documentation/, .DS_Store) after cleaning any .git contamination in destination` |
| 6 (skill) | `would zip claude-code-skill/ into <name>.skill and copy to <marketplace>/skills/<name>/<name>.skill` |
| 7a | `would update marketplace.json entry "<name>" → version <new-version>` (plugins only) |
| 7b | `would regenerate marketplace index with <name> v<new-version>` |
| 8a | `would commit "publish: <name> v<new-version> — <summary>" in <marketplace>` |
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
| 2     | Version Bump Prompt         | PASS   | would prompt for bump type (selected: <type> → <new-version>) |
| 0     | PREFLIGHT                   | PASS   | would run 4 checks (<status>)                         |
| 3     | Changelog Summary Prompt    | PASS   | would prompt for summary (captured)                   |
| 4     | Write Version Bump          | PASS   | would write version <new-version> to <manifest-file>  |
| 5     | Write CHANGELOG Entry       | PASS   | would prepend v<new-version> entry to <container>/CHANGELOG.md |
| 6     | Package                     | PASS   | would rsync <plugins-dir>/<name>/ to <marketplace>/plugins/<name>/ (excluding .git/, .planning/, .claude/, documentation/, .DS_Store) after cleaning .git contamination / would zip claude-code-skill/ into <name>.skill and copy to <marketplace>/skills/<name>/<name>.skill |
| 7     | Copy to Marketplace + Index | PASS   | would update marketplace.json entry and regenerate index with <name> v<new-version> |
| 8     | Git Commit and Push         | PASS   | would commit and push to origin/main                  |
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
