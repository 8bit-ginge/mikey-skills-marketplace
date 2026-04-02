# Validation Rules

Complete check definitions for all validation categories. This is the single source of truth for every check the `/marketplace:validate` command performs. All fix message templates and pass/fail criteria are defined here.

## Table of Contents

1. [Ecosystem Paths](#ecosystem-paths)
2. [Type Detection (VAL-02)](#type-detection-val-02)
3. [YAML Frontmatter Parsing (VAL-08)](#yaml-frontmatter-parsing-val-08)
4. [Check Categories](#check-categories)
   - [Category 1: Container Structure (VAL-03)](#category-1-container-structure-val-03)
   - [Category 2: SKILL.md Frontmatter (VAL-04, VAL-08)](#category-2-skillmd-frontmatter-val-04-val-08)
   - [Category 3: Plugin JSON (VAL-05)](#category-3-plugin-json-val-05)
   - [Category 4: Documentation (VAL-06)](#category-4-documentation-val-06)
5. [Output Format Contract (D-03)](#output-format-contract-d-03)
6. [Full Scan Mode (VAL-07)](#full-scan-mode-val-07)

---

## Ecosystem Paths

These absolute paths are the ground truth for locating skills and plugins. Do NOT embed these in SKILL.md body — they live here only.

- **Ecosystem root:** `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/`
- **Skills directory:** `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/`
- **Plugins directory:** `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-plugins/`
- **Marketplace repo:** `/Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/`

---

## Type Detection (VAL-02)

Given a name from `$ARGUMENTS`, determine whether it is a skill or plugin:

1. Check if `<skills-dir>/<name>/` exists as a directory — if yes, type is `skill`
2. Check if `<plugins-dir>/<name>/` exists as a directory — if yes, type is `plugin`
3. If both exist, treat as `skill` (skills take precedence)
4. If neither exists, report: `Artifact '<name>' not found in claude-skills/ or claude-plugins/`

Use Bash to check directory existence:

```bash
[ -d "/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/<name>" ] && echo "skill"
[ -d "/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-plugins/<name>" ] && echo "plugin"
```

---

## YAML Frontmatter Parsing (VAL-08)

Apply these steps when parsing any SKILL.md file to isolate frontmatter from body content. This prevents false positives from body content containing YAML-like keys.

**Parsing procedure:**

1. Read the entire SKILL.md file content
2. Find the first line that is exactly `---` (no leading/trailing whitespace except newline)
3. Find the next line that is exactly `---` — this is the closing delimiter
4. All content between these two delimiters is the **frontmatter block**
5. Apply ALL frontmatter checks ONLY to this extracted block
6. Body line count starts from the line AFTER the closing `---` delimiter

**Block scalar descriptions (recipe-optimiser edge case):**

YAML supports block scalar format for multi-line strings. When the `description:` value uses block scalar format, it looks like:

```yaml
description: |
  First line of description
  Second line of description
  Third line of description
```

When you encounter this format:
- Collect all indented continuation lines following `description: |`
- Strip the leading whitespace from each continuation line
- Concatenate into a single string (with spaces between lines if needed)
- Use the concatenated string for character counting and XML bracket checks

**Body line count:**

Count lines starting from the line immediately after the closing `---` delimiter to the last line of the file. Empty lines count. The limit is 500 lines.

---

## Check Categories

### Category 1: Container Structure (VAL-03)

#### For skills

The container is the top-level directory at `<skills-dir>/<name>/`.

| Check | Pass Condition | Fail Message Template |
|-------|---------------|----------------------|
| Folder exists | `<skills-dir>/<name>/` is a directory | `Folder missing at claude-skills/<name>/` |
| Source folder present | `<skills-dir>/<name>/claude-code-skill/` exists | `claude-code-skill/ missing — create claude-skills/<name>/claude-code-skill/` |
| SKILL.md present | `<skills-dir>/<name>/claude-code-skill/SKILL.md` exists | `SKILL.md missing — create claude-skills/<name>/claude-code-skill/SKILL.md` |
| No README inside source | `<skills-dir>/<name>/claude-code-skill/README.md` does NOT exist | `README.md found inside claude-code-skill/ — move to container level or delete` |
| Kebab-case name | Folder name matches `^[a-z0-9][a-z0-9-]*[a-z0-9]$` or single char `^[a-z0-9]$` | `Folder name '<name>' is not kebab-case — rename to use only lowercase letters, numbers, and hyphens` |
| Desktop folder present | `<skills-dir>/<name>/claude-desktop-skill/` exists | `claude-desktop-skill/ missing — create claude-skills/<name>/claude-desktop-skill/` |

**Pass message pattern for each check:** `<check description>` (on same line with checkmark prefix)

#### For plugins

The container is the top-level directory at `<plugins-dir>/<name>/`.

| Check | Pass Condition | Fail Message Template |
|-------|---------------|----------------------|
| Folder exists | `<plugins-dir>/<name>/` is a directory | `Folder missing at claude-plugins/<name>/` |
| Plugin manifest present | `<plugins-dir>/<name>/.claude-plugin/plugin.json` exists | `.claude-plugin/plugin.json missing — create it with name, description, version, author fields` |
| Kebab-case name | Folder name matches `^[a-z0-9][a-z0-9-]*[a-z0-9]$` or single char `^[a-z0-9]$` | `Folder name '<name>' is not kebab-case — rename to use only lowercase letters, numbers, and hyphens` |

---

### Category 2: SKILL.md Frontmatter (VAL-04, VAL-08)

**Applies to skills only.** Skip this category entirely for plugins.

Parse frontmatter using the VAL-08 procedure above before running any of these checks.

| Check | Pass Condition | Fail Message Template |
|-------|---------------|----------------------|
| YAML block present | Frontmatter block found between `---` delimiters | `No YAML frontmatter found — add --- delimited block at top of SKILL.md` |
| name field present | `name:` key exists in frontmatter block | `name field missing in frontmatter — add name: <name>` |
| name matches folder | name field value equals the skill's folder name | `name mismatch: folder is '<folder>' but name field is '<value>' — update name to '<folder>'` |
| description present | `description:` key exists in frontmatter block | `description missing — add a description under 1024 chars` |
| description length | Character count of description value is less than or equal to 1024 | `description is <count>/1024 chars — trim <excess> characters to fit within limit` |
| description budget (pass) | When description passes length check | `description is <count>/1024 chars — <remaining> chars remaining` |
| No XML angle brackets | description value contains neither `<` nor `>` | `description contains XML angle brackets — remove all < and > characters` |
| Body line count | Lines after closing `---` delimiter are less than or equal to 500 | `body is <count>/500 lines — reduce by <excess> lines` |
| Body line count (pass) | When body passes | `body is <count>/500 lines` |

**Template variable meanings:**
- `<count>` — actual measured character or line count
- `<excess>` — how many over the limit (count minus limit)
- `<remaining>` — how many under the limit (limit minus count)

---

### Category 3: Plugin JSON (VAL-05)

**Applies to plugins only.** Skip this category entirely for skills.

| Check | Pass Condition | Fail Message Template |
|-------|---------------|----------------------|
| Valid JSON | `python3 -c "import json; json.load(open('<path>'))"` succeeds without error | `plugin.json is not valid JSON — fix syntax error: <error message>` |
| name field present | `name` key exists in parsed JSON | `plugin.json: name field missing — add "name": "<name>"` |
| name is kebab-case | name value matches `^[a-z0-9][a-z0-9-]*[a-z0-9]$` or single word `^[a-z0-9]+$` | `plugin.json: name '<value>' is not kebab-case — use only lowercase letters, numbers, and hyphens` |
| version field present | `version` key exists in parsed JSON | `plugin.json: version field missing — add "version": "1.0.0"` |
| Semver format | version value matches `^[0-9]+\.[0-9]+\.[0-9]+$` | `plugin.json: version '<value>' is not semver — use format X.X.X (e.g., 1.0.0)` |

---

### Category 4: Documentation (VAL-06)

**Applies to both skills and plugins.**

| Check | Pass Condition | Fail Message Template |
|-------|---------------|----------------------|
| README.md present | `<container-dir>/README.md` exists | `No README.md at container level — create <container-path>/README.md` |
| CHANGELOG.md present | `<container-dir>/CHANGELOG.md` exists | `No CHANGELOG.md at container level — create <container-path>/CHANGELOG.md` |

Where `<container-dir>` is:
- For skills: `claude-skills/<name>/` (i.e., `<skills-dir>/<name>/`)
- For plugins: `claude-plugins/<name>/` (i.e., `<plugins-dir>/<name>/`)

---

## Output Format Contract (D-03)

Use this exact format for every validation report. This is the visual contract — do not deviate from the structure.

```
## validate: <name>

### Container
[check lines with checkmark or cross prefix — one per check]

### SKILL.md Frontmatter   (for skills)
[check lines with checkmark or cross prefix — one per check]

### Plugin JSON   (for plugins — replaces SKILL.md Frontmatter section)
[check lines with checkmark or cross prefix — one per check]

### Documentation
[check lines with checkmark or cross prefix — one per check]

---
Result: <N> failures   (or "Result: all checks passed" when N=0)
```

**Check line format:**
- Pass: `checkmark <description>`
- Fail: `cross <description>`
  - `  Fix: <fix message from template>` (indented on next line, only for failures)

**Full example output:**

```
## validate: recipe-optimiser

### Container
checkmark Folder exists at claude-skills/recipe-optimiser/
checkmark claude-code-skill/ present
checkmark SKILL.md present
checkmark No README.md inside claude-code-skill/
checkmark Folder name is kebab-case
checkmark claude-desktop-skill/ present

### SKILL.md Frontmatter
checkmark name: recipe-optimiser (matches folder)
checkmark description present
checkmark description is 877/1024 chars — 147 chars remaining
checkmark No XML angle brackets in description
checkmark body is 145/500 lines

### Documentation
checkmark README.md present at container level
checkmark CHANGELOG.md present at container level

---
Result: all checks passed
```

**Failure example:**

```
## validate: interview-research

### Container
checkmark Folder exists at claude-skills/interview-research/
checkmark claude-code-skill/ present
checkmark SKILL.md present
checkmark No README.md inside claude-code-skill/
checkmark Folder name is kebab-case
checkmark claude-desktop-skill/ present

### SKILL.md Frontmatter
checkmark name: interview-research (matches folder)
checkmark description present
checkmark description is 726/1024 chars — 298 chars remaining
checkmark No XML angle brackets in description
checkmark body is 228/500 lines

### Documentation
cross No README.md at container level — create claude-skills/interview-research/README.md
cross No CHANGELOG.md at container level — create claude-skills/interview-research/CHANGELOG.md

---
Result: 2 failures
```

**Note:** Use actual Unicode checkmark (✅) and cross (❌) characters in rendered output.

---

## Full Scan Mode (VAL-07)

When `$ARGUMENTS` is empty, run a full ecosystem scan:

**Procedure:**

1. List all entries in `<skills-dir>/` — filter to directories only; skip any directory whose name starts with `.` (hidden dirs, `.DS_Store`)
2. List all entries in `<plugins-dir>/` — filter to directories only; skip any directory whose name starts with `.`
3. For each skill directory: run all applicable check categories (Container, SKILL.md Frontmatter, Documentation)
4. For each plugin directory: run all applicable check categories (Container, Plugin JSON, Documentation)
5. Output the per-artifact detailed report (D-03 format) for each artifact in sequence
6. After ALL individual reports, output the summary table

**Bash to list valid subdirectories (excluding hidden):**

```bash
ls -d /Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-skills/*/  2>/dev/null | xargs -I{} basename {} | grep -v '^\.'
```

**Summary table format (output after all individual reports):**

```
## Ecosystem Validation Summary

| Artifact | Type | Result | Failures |
|----------|------|--------|----------|
| recipe-optimiser | skill | Pass | 0 |
| interview-research | skill | Fail | 2 |
| prompt-engineer | skill | Fail | 2 |
| skill-forge | plugin | Fail | 2 |
| skill-marketplace-manager | plugin | Fail | 3 |
```

Use "Pass" or "Fail" in the Result column. Failures column is the count of failed checks for that artifact.

---

*This file is the complete check definition reference. SKILL.md loads this file explicitly via `Read: ${CLAUDE_SKILL_DIR}/references/validation-rules.md` before running any checks.*
