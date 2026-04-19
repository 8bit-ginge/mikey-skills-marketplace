# Scaffold Templates: New Plugin

Complete template content, ecosystem paths, guard check logic, directory creation order, confirmation output format, and silent validation checks for the `/marketplace-manager:new-plugin` command. This is the single source of truth for all plugin scaffold details.

## Table of Contents

1. [Ecosystem Paths](#ecosystem-paths)
2. [Guard Checks (SCAF-03 + SCAF-04)](#guard-checks-scaf-03--scaf-04)
3. [Plugin Container Template](#plugin-container-template)
4. [Directory Creation Order](#directory-creation-order)
5. [Confirmation Output (D-04)](#confirmation-output-d-04)
6. [Silent Validation (SCAF-05)](#silent-validation-scaf-05)

---

## Ecosystem Paths

These absolute paths are the ground truth for locating plugins on this machine. Do NOT embed these in SKILL.md body — they live here only.

- **Plugins directory:** `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/`

Use `PLUGINS_DIR` as the variable name when referencing this path in Bash operations:

```bash
PLUGINS_DIR="/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins"
```

---

## Guard Checks (SCAF-03 + SCAF-04)

Both guards MUST pass before any file or directory creation. Run them in this order.

### Guard 1 — Empty Argument Check

Before any name validation, check if `$ARGUMENTS` is empty:

```bash
NAME="$ARGUMENTS"
if [ -z "$NAME" ]; then
  echo "Error: No name provided. Usage: /marketplace-manager:new-plugin [name]"
  # STOP — do not proceed
fi
```

On failure output:
```
Error: No name provided. Usage: /marketplace-manager:new-plugin [name]
```

### Guard 2 — Name Validation (SCAF-03)

Validate name using kebab-case regex. Accept single characters and multi-character kebab-case names:

```bash
# Regex: ^[a-z0-9]([a-z0-9-]*[a-z0-9])?$
# Matches: "my-plugin", "plugin", "my-plugin-v2", "a"
# Rejects: "My Plugin", "my_plugin", "my plugin", "-plugin", "plugin-", "my--plugin"
echo "$NAME" | grep -qE '^[a-z0-9]([a-z0-9-]*[a-z0-9])?$' || {
  echo "Error: '$NAME' is not a valid kebab-case name. Use only lowercase letters, numbers, and hyphens. Cannot start or end with a hyphen."
  # STOP — do not proceed
}
```

On failure output:
```
Error: 'NAME' is not a valid kebab-case name. Use only lowercase letters, numbers, and hyphens. Cannot start or end with a hyphen.
```

**CRITICAL:** No files or directories may be created before this check passes.

### Guard 3 — Existence Check (SCAF-04)

Check if a plugin container already exists at the target location:

```bash
PLUGINS_DIR="/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins"
if [ -d "$PLUGINS_DIR/$NAME" ]; then
  echo "Error: Plugin '$NAME' already exists at plugins/$NAME/"
  # STOP — do not proceed
fi
```

On failure output:
```
Error: Plugin 'NAME' already exists at plugins/NAME/
```

**CRITICAL:** No files or directories may be created before this check passes.

---

## Plugin Container Template

After both guards pass, create these files and directories. Replace `NAME` with the actual value from `$ARGUMENTS` in all file contents.

### File 1 — `.claude-plugin/plugin.json` (D-02)

Path: `plugins/NAME/.claude-plugin/plugin.json`

Content (replace NAME with actual plugin name):
```json
{
  "name": "NAME",
  "version": "1.0.0",
  "description": "TODO: Describe this plugin.",
  "author": "TODO: Your name"
}
```

Notes:
- `author` is a plain string "TODO: Your name" — this differs from the existing marketplace plugin.json which uses an object format. D-02 was explicitly confirmed by the user — use the plain string format for scaffolded plugins.
- `version` always starts at `1.0.0`
- Replace `NAME` with the actual plugin name from `$ARGUMENTS`

### File 2 — `README.md` (D-05)

Path: `plugins/NAME/README.md`

Content (replace NAME with actual plugin name):
```markdown
# NAME

TODO: One-line description of this plugin.

## Usage

TODO: Describe how to use this plugin.

## Requirements

- Claude Code 1.0.33+
```

### File 3 — `CHANGELOG.md` (D-05)

Path: `plugins/NAME/CHANGELOG.md`

Content (no substitution needed — CHANGELOG content is the same for any new plugin):
```markdown
# Changelog

<!--
  User-facing changelog — one short sentence per release.
  Syncs to the marketplace copy on every publish (Stage 6 rsync).
  Dev history (milestones, phases, decisions) lives in CHANGELOG-internal.md.
-->

## [Unreleased]

- Initial scaffold
```

### File 3b — `CHANGELOG-internal.md` (D-05)

Path: `plugins/NAME/CHANGELOG-internal.md`

Content (no substitution needed — CHANGELOG-internal content is the same for any new plugin):
```markdown
# Changelog (Internal)

<!--
  Dev-internal changelog — milestones, phases, decisions, refactor notes.
  Excluded from rsync (Stage 6). Never shipped to the marketplace.
  User-facing entries live in CHANGELOG.md.
-->

## [Unreleased]

- Initial scaffold
```

### Directory 1 — `documentation/`

Path: `plugins/NAME/documentation/`

Create as empty directory. Not required by validate but part of the plugin scaffold spec (SCAF-02).

### Directory 2 — `commands/`

Path: `plugins/NAME/commands/`

Create as empty directory. Not required by validate but part of the plugin scaffold spec (SCAF-02).

### Directory 3 — `skills/`

Path: `plugins/NAME/skills/`

Create as empty directory. Not required by validate but part of the plugin scaffold spec (SCAF-02).

---

## Directory Creation Order

After both guards pass, execute in this exact order:

1. `mkdir -p $PLUGINS_DIR/NAME/.claude-plugin/`
2. `mkdir -p $PLUGINS_DIR/NAME/documentation/`
3. `mkdir -p $PLUGINS_DIR/NAME/commands/`
4. `mkdir -p $PLUGINS_DIR/NAME/skills/`
5. Write `plugin.json` with NAME substituted for actual plugin name
6. Write `README.md` with NAME substituted for actual plugin name
7. Write `CHANGELOG.md` (no substitution needed)
8. Write `CHANGELOG-internal.md` (no substitution needed)

Step 1 implicitly creates the container directory `plugins/NAME/` — no separate mkdir needed for the container.

---

## Confirmation Output (D-04)

After all files are created successfully, output exactly this format (replace NAME with actual plugin name):

```
Created: NAME/
  .claude-plugin/
    plugin.json
  README.md
  CHANGELOG.md
  CHANGELOG-internal.md
  documentation/
  commands/
  skills/

✔ validate: NAME — all checks passed
```

The validate summary line is always on a separate line after a blank line. The file tree uses 2-space indentation.

---

## Silent Validation (SCAF-05)

After file creation and before outputting the confirmation, run these inline validation checks. These mirror the checks from `skills/validate/references/validation-rules.md` Category 1 (Container), Category 3 (Plugin JSON), and Category 4 (Documentation) for plugins.

Run each check with Bash. If ALL pass, output the confirmation format above. If ANY fail, output the full failure report using the same format as `/marketplace-manager:validate`.

### Check 1 — Folder exists (Category 1)

```bash
[ -d "$PLUGINS_DIR/$NAME" ] && echo "PASS" || echo "FAIL: Folder missing at plugins/$NAME/"
```

Pass condition: `$PLUGINS_DIR/$NAME/` is a directory.

### Check 2 — Plugin manifest present (Category 1)

```bash
[ -f "$PLUGINS_DIR/$NAME/.claude-plugin/plugin.json" ] && echo "PASS" || echo "FAIL: .claude-plugin/plugin.json missing"
```

Pass condition: `$PLUGINS_DIR/$NAME/.claude-plugin/plugin.json` exists as a file.

### Check 3 — Kebab-case name (Category 1)

Already validated by Guard 2 before creation — this check always passes for a freshly scaffolded container.

### Check 4 — Valid JSON (Category 3)

```bash
python3 -c "import json; json.load(open('$PLUGINS_DIR/$NAME/.claude-plugin/plugin.json'))" && echo "PASS" || echo "FAIL: plugin.json is not valid JSON"
```

Pass condition: python3 JSON parse succeeds without error.

### Check 5 — name field present and matches folder (Category 3)

```bash
python3 -c "
import json
data = json.load(open('$PLUGINS_DIR/$NAME/.claude-plugin/plugin.json'))
assert 'name' in data, 'name field missing'
assert data['name'] == '$NAME', f'name mismatch: folder is $NAME but name is {data[\"name\"]}'
print('PASS')
"
```

Pass condition: `name` key exists and its value equals the folder name.

### Check 6 — version field in semver format (Category 3)

```bash
python3 -c "
import json, re
data = json.load(open('$PLUGINS_DIR/$NAME/.claude-plugin/plugin.json'))
assert 'version' in data, 'version field missing'
assert re.match(r'^[0-9]+\.[0-9]+\.[0-9]+$', data['version']), f'version not semver: {data[\"version\"]}'
print('PASS')
"
```

Pass condition: `version` key exists and matches `^[0-9]+\.[0-9]+\.[0-9]+$`.

### Check 7 — README.md present (Category 4)

```bash
[ -f "$PLUGINS_DIR/$NAME/README.md" ] && echo "PASS" || echo "FAIL: No README.md at container level"
```

Pass condition: `$PLUGINS_DIR/$NAME/README.md` exists as a file.

### Check 8 — CHANGELOG.md present (Category 4)

```bash
[ -f "$PLUGINS_DIR/$NAME/CHANGELOG.md" ] && echo "PASS" || echo "FAIL: No CHANGELOG.md at container level"
```

Pass condition: `$PLUGINS_DIR/$NAME/CHANGELOG.md` exists as a file.

### Check 9 — CHANGELOG-internal.md present (Category 4)

```bash
[ -f "$PLUGINS_DIR/$NAME/CHANGELOG-internal.md" ] && echo "PASS" || echo "FAIL: No CHANGELOG-internal.md at container level"
```

Pass condition: `$PLUGINS_DIR/$NAME/CHANGELOG-internal.md` exists as a file.

### Implementation Note

When aggregating check results with counters in bash, use `PASS=$((PASS + 1))` and `FAIL=$((FAIL + 1))`. Do NOT use `((PASS++))` — it returns exit code 1 when incrementing from 0, which causes false failures.

### Validation Result

If ALL 9 checks pass:
- Output the confirmation format from the Confirmation Output section above
- The `✔ validate: NAME — all checks passed` line is the final output

If ANY check fails:
- Output the full failure report using the D-03 format from validation-rules.md
- Note: For a freshly scaffolded container, validation should always pass. Any failure indicates a bug in the scaffold template itself.

---

*This file is the complete scaffold reference for the new-plugin command. SKILL.md loads this file explicitly via `Read: ${CLAUDE_SKILL_DIR}/references/scaffold-templates.md` before executing.*
