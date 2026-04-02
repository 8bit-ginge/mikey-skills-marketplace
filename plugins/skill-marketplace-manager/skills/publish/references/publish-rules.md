# Publish Rules

Complete pipeline logic for the `/marketplace:publish` command. This is the single source of truth for all stage definitions, ecosystem paths, packaging commands, output contracts, and failure handling. SKILL.md loads this file via explicit Read instruction and delegates all execution here.

## Table of Contents

1. [Ecosystem Paths](#ecosystem-paths)
2. [Type Detection](#type-detection)
3. [Pipeline Overview](#pipeline-overview)
4. [Stage 1: Validation Gate](#stage-1-validation-gate)
5. [Stage 2: Version Bump Prompt](#stage-2-version-bump-prompt)
6. [Stage 3: Changelog Summary Prompt](#stage-3-changelog-summary-prompt)
7. [Stage 4: Write Version Bump](#stage-4-write-version-bump)
8. [Stage 5: Write CHANGELOG Entry](#stage-5-write-changelog-entry)
9. [Stage 6: Package](#stage-6-package)
10. [Stage 7: Copy to Marketplace and Update Index](#stage-7-copy-to-marketplace-and-update-index)
11. [Stage 8: Git Commit and Push](#stage-8-git-commit-and-push)
12. [Output Format Contracts](#output-format-contracts)
13. [Failure Handling](#failure-handling)

---

## Ecosystem Paths

These absolute paths are the ground truth for locating skills, plugins, and the marketplace. Do NOT embed these in SKILL.md body — they live here only.

- **Ecosystem root:** `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/`
- **Skills directory:** `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/`
- **Plugins directory:** `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-plugins/`
- **Marketplace repo:** `/Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/`
- **Marketplace remote:** `https://github.com/8bit-ginge/mikey-skills-marketplace.git`

Abbreviated references used throughout this file:
- `<skills-dir>` = `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/`
- `<plugins-dir>` = `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-plugins/`
- `<marketplace>` = `/Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/`

---

## Type Detection

Given a name from `$ARGUMENTS`, determine whether it is a skill or plugin. Use the same logic as validation-rules.md.

1. Check if `<skills-dir>/<name>/` exists as a directory — if yes, type is `skill`
2. Check if `<plugins-dir>/<name>/` exists as a directory — if yes, type is `plugin`
3. If both exist, treat as `skill` (skills take precedence)
4. If neither exists, report: `Artifact '<name>' not found in claude-skills/ or claude-plugins/` and stop

```bash
[ -d "/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/<name>" ] && echo "skill"
[ -d "/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-plugins/<name>" ] && echo "plugin"
```

---

## Pipeline Overview

Eight sequential stages. Each stage is gated — no stage starts until the previous succeeds. If any stage fails, the pipeline stops immediately and shows the failure output defined in the Failure Handling section.

| Stage | Description | Writes? |
|-------|-------------|---------|
| 1 | Validation Gate | No |
| 2 | Version Bump Prompt | No |
| 3 | Changelog Summary Prompt | No |
| 4 | Write Version Bump | Yes |
| 5 | Write CHANGELOG Entry | Yes |
| 6 | Package | Yes |
| 7 | Copy to Marketplace + Update Index | Yes |
| 8 | Git Commit and Push | Yes |

**Stages 1-3 are read-only.** Stages 4-8 write files. The validation gate (Stage 1) ensures no writes occur if the artifact has issues. The two prompts (Stages 2-3) collect all user input before any writes begin.

---

## Stage 1: Validation Gate

**Requirement:** PUB-02 — hard stop if any checks fail before any writes occur.

Run the full validation logic from `validation-rules.md` against the artifact. To do this:

1. Read `skills/validate/references/validation-rules.md` (from the plugin root — the absolute path is alongside this plugin's skills/ directory)
2. Execute ALL applicable check categories for the detected artifact type:
   - For skills: Container Structure, SKILL.md Frontmatter, Documentation
   - For plugins: Container Structure, Plugin JSON, Documentation
3. Tally the total number of failures

**If ANY check fails:**
- Stop the pipeline immediately — do NOT proceed to Stage 2
- Display the full validation report in the standard D-03 format (same output as `/marketplace:validate`)
- After the report, append:
  ```
  Fix the issues above, then run `/marketplace:publish <name>` again.
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
content = open('/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/<name>/claude-code-skill/SKILL.md').read()
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
d = json.load(open('/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-plugins/<name>/.claude-plugin/plugin.json'))
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

### For skills — update SKILL.md frontmatter

Read the SKILL.md file, locate the `version:` line within the `---` delimited frontmatter block only, replace with new version. Use python3 for safe replacement.

```python
import re

skill_md_path = '/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/<name>/claude-code-skill/SKILL.md'
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

plugin_json_path = '/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-plugins/<name>/.claude-plugin/plugin.json'
new_version = '<new-version>'

with open(plugin_json_path) as f:
    data = json.load(f)

data['version'] = new_version

with open(plugin_json_path, 'w') as f:
    json.dump(data, f, indent=2)
    f.write('\n')
```

**IMPORTANT — progressive output:** Immediately after writing the version, print the tick-mark line to the user. Do NOT wait until later stages to show this. Print it now:

`✅ Version bumped: <old-version> → <new-version>`

---

## Stage 5: Write CHANGELOG Entry

**Requirement:** PUB-01 — insert new entry at the top of CHANGELOG.md.

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
changelog_path = '/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-<type>s/<name>/CHANGELOG.md'
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

with open(changelog_path, 'w') as f:
    f.write(new_content)
```

**IMPORTANT — progressive output:** Immediately after writing the changelog entry, print the tick-mark line to the user. Do NOT batch this with later stage outputs. Print it now:

`✅ CHANGELOG.md updated`

---

## Stage 6: Package

### For skills (PUB-04) — ZIP with SKILL.md at root

**Critical:** Always `cd` into `claude-code-skill/` before running zip. If you run zip from outside that directory, the ZIP will contain `claude-code-skill/SKILL.md` instead of `SKILL.md` at root — Claude Desktop will fail to load the skill.

```bash
# Step 1: Rebuild the .skill ZIP
mkdir -p /Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/<name>/claude-desktop-skill/
cd /Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/<name>/claude-code-skill && zip -r ../claude-desktop-skill/<name>.skill . -x "evals/*" "*.pyc" "__pycache__/*" ".DS_Store"

# Step 2: Create marketplace destination and copy .skill file
mkdir -p /Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/
cp /Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/<name>/claude-desktop-skill/<name>.skill /Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill

# Step 3: Verify — SKILL.md must appear at root, NOT nested under a folder
unzip -l /Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill
```

If the verify step shows `SKILL.md` nested under a directory prefix (e.g., `claude-code-skill/SKILL.md`), the package is invalid — stop with a mid-pipeline failure (D-08).

Read the file size for the output line:

```bash
ls -lh /Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill | awk '{print $5}'
```

After packaging, print: `✅ Packaged: <name>.skill (<size>)`

### For plugins (PUB-05) — rsync copy with exclusions

```bash
rsync -a \
  --exclude='documentation/' \
  --exclude='.DS_Store' \
  /Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-plugins/<name>/ \
  /Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/plugins/<name>/
```

After copying, print: `✅ Packaged: <name> (rsync copy)`

---

## Stage 7: Copy to Marketplace and Update Index

### Sub-step A: Update marketplace.json (plugins only — PUB-06)

**Skills NEVER get marketplace.json entries.** Only plugins are registered in `marketplace.json`.

For plugins, use the read-parse-modify-rewrite pattern:

```python
import json

marketplace_json_path = '/Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/.claude-plugin/marketplace.json'
name = '<plugin-name>'
new_version = '<new-version>'
new_description = '<description from plugin.json>'

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
ls /Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/ 2>/dev/null
```

For each skill directory found, read version and description from the `.skill` ZIP:

```bash
# Read version from .skill ZIP
unzip -p /Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill "SKILL.md" 2>/dev/null | python3 -c "
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
unzip -p /Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill "SKILL.md" 2>/dev/null | python3 -c "
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
ls /Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/plugins/ 2>/dev/null
```

For each plugin directory found, read version and description from `.claude-plugin/plugin.json`:

```python
import json
d = json.load(open('/Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/plugins/<name>/.claude-plugin/plugin.json'))
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
*Auto-generated by skill-marketplace-manager*
```

Write to `/Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/README.md`.

If there are no published skills yet, omit the Skills table rows (keep the header). If there are no published plugins yet, omit the Plugins table rows (keep the header).

After writing, print: `✅ README.md index regenerated`

---

## Stage 8: Git Commit and Push

**Requirements:** PUB-08, PUB-09

```bash
MARKETPLACE=/Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace

# Stage all changes in marketplace repo
git -C "$MARKETPLACE" add .

# Commit with structured message
git -C "$MARKETPLACE" commit -m "publish: <name> v<new-version> — <summary>"

# Push and capture exit code
git -C "$MARKETPLACE" push origin main
PUSH_EXIT=$?
```

If `PUSH_EXIT` is 0 (success):
- Print: `✅ Committed and pushed: publish: <name> v<new-version> — <summary>`
- Then print the final success line (D-02):
  ```
  ✔ Published <name> v<new-version>
  ```
- Pipeline complete.

If `PUSH_EXIT` is not 0 (failure):
- Show the D-10 failure format (see Failure Handling section)
- Do NOT rollback local changes — all local writes are preserved
- Do NOT attempt retry

---

## Output Format Contracts

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
```

The `✔` on the final line uses the heavy check mark (U+2714) to distinguish the completion summary from the per-stage tick marks (U+2705).

For skills, the packaged line shows: `✅ Packaged: <name>.skill (<size>)`
For plugins, the packaged line shows: `✅ Packaged: <name> (rsync copy)`

---

## Failure Handling

### Scenario A: Validation Gate Failure (D-09)

Triggered when Stage 1 finds any failed checks.

Display the full validation report in standard D-03 format (same as `/marketplace:validate` output), then append:

```
Fix the issues above, then run `/marketplace:publish <name>` again.
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
  cd /Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/<name>/claude-code-skill && zip -r ../claude-desktop-skill/<name>.skill . -x "evals/*" "*.pyc" "__pycache__/*" ".DS_Store"
```

---

### Scenario C: Git Push Failure (D-10)

Triggered when `git push` returns a non-zero exit code.

Show all completed stages with ✅, then:

```
❌ Git push failed

**What happened:** GitHub rejected the upload —
usually this means someone else pushed changes
that you don't have yet.

**Your changes are safe** — everything is saved
locally. Nothing was lost.

**To fix:** Run this command in your terminal:
  git -C /Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace push origin main
```

Do NOT attempt rollback. Do NOT retry automatically. The user runs the recovery command when ready.

---

*This file is the complete pipeline logic reference. SKILL.md loads this file explicitly via `Read: ${CLAUDE_SKILL_DIR}/references/publish-rules.md` before executing any pipeline stage.*
