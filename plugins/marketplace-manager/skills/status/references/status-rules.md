# Status Rules

Complete lookup procedures, ecosystem paths, card format templates, and status label definitions for the `/marketplace-manager:status` command. This is the single source of truth for everything the status command needs.

## Table of Contents

1. [Ecosystem Paths](#ecosystem-paths)
2. [Artifact Discovery](#artifact-discovery)
3. [Version Reading](#version-reading)
4. [Marketplace Lookup](#marketplace-lookup)
5. [Cache Lookup](#cache-lookup)
6. [Version Normalisation](#version-normalisation)
7. [Status Label Logic](#status-label-logic)
8. [Card Format Template](#card-format-template)
9. [Git Divergence Footer](#git-divergence-footer)
10. [Summary Footer](#summary-footer)
11. [Brief Mode](#brief-mode)
12. [Full Output Structure](#full-output-structure)

---

## Ecosystem Paths

These absolute paths are the ground truth for locating skills and plugins. Do NOT embed these in SKILL.md body — they live here only.

- **Ecosystem root:** `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/`
- **Skills directory:** `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/`
- **Plugins directory:** `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/`
- **Marketplace repo:** `/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/`
- **Claude Code install cache:** `~/.claude/plugins/installed_plugins.json`
- **Marketplace name:** `mikey-skills` (used for cache key construction: `plugin-name@mikey-skills`)
- **Marketplace skills path convention:** `<marketplace-repo>/skills/<name>/<name>.skill` (not yet populated — Phase 4 will publish here)

---

## Artifact Discovery

Use these Bash commands to find all skills and plugins in the ecosystem. Skip hidden directories (those starting with `.`).

```bash
# List skill names (skip hidden dirs)
ls -d /Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/*/  2>/dev/null | xargs -I{} basename {} | grep -v '^\.'

# List plugin names (skip hidden dirs)
ls -d /Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/*/  2>/dev/null | xargs -I{} basename {} | grep -v '^\.'
```

Sort all artifacts alphabetically by name before rendering cards.

---

## Version Reading

Two procedures — one for skills (YAML frontmatter), one for plugins (JSON).

### For Skills — YAML Frontmatter Delimiter Isolation

Read version from the first SKILL.md found among the skill's candidate layouts. Try each path below in order and stop at the first match. This prevents the reader from returning `MISSING` when a skill stores its SKILL.md in a nested or flat layout rather than the canonical `claude-code-skill/` subdirectory. The frontmatter delimiter isolation approach prevents false positives from body content containing YAML-like keys.

**Candidate paths (first match wins):**

1. `<skills-dir>/<name>/claude-code-skill/SKILL.md` — canonical layout (interview-research, prompt-engineer)
2. `<skills-dir>/<name>/skills/<name>/SKILL.md` — nested-bundle layout (recipe-optimiser)
3. `<skills-dir>/<name>/SKILL.md` — flat layout (future-proofing)

**Return contract:**

- `<version>` — file found, frontmatter has a `version:` key
- `NO_VERSION` — file found, frontmatter exists but no `version:` key
- `MISSING` — no candidate path contains a SKILL.md

```bash
python3 -c "
import re, os
name = '<name>'
base = '/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills'
candidates = [
    f'{base}/{name}/claude-code-skill/SKILL.md',
    f'{base}/{name}/skills/{name}/SKILL.md',
    f'{base}/{name}/SKILL.md',
]
for path in candidates:
    if os.path.isfile(path):
        content = open(path).read()
        m = re.search(r'^---\n(.*?)^---', content, re.MULTILINE | re.DOTALL)
        if m:
            vm = re.search(r'^version:\s*(.+)$', m.group(1), re.MULTILINE)
            print(vm.group(1).strip() if vm else 'NO_VERSION')
        else:
            print('NO_VERSION')
        break
else:
    print('MISSING')
"
```

Replace `<name>` with the actual skill name. If no candidate path contains a SKILL.md, treat as `MISSING`.

### For Plugins — JSON Parse

Read version from `<plugins-dir>/<name>/.claude-plugin/plugin.json`:

```bash
python3 -c "
import json
d = json.load(open('/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/<name>/.claude-plugin/plugin.json'))
print(d.get('version', 'MISSING'))
"
```

Replace `<name>` with the actual plugin name. If the file does not exist, treat as `MISSING`.

---

## Marketplace Lookup

Two procedures — one for skills (ZIP file), one for plugins (directory).

### For Skills — Check .skill ZIP Existence then Read Version

```bash
# Check existence
[ -f "/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill" ] && echo "present" || echo "absent"

# Read version from .skill ZIP (if present)
unzip -p "/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/skills/<name>/<name>.skill" "SKILL.md" 2>/dev/null | python3 -c "
import re, sys
content = sys.stdin.read()
m = re.search(r'^---\n(.*?)^---', content, re.MULTILINE | re.DOTALL)
if m:
    vm = re.search(r'^version:\s*(.+)$', m.group(1), re.MULTILINE)
    print(vm.group(1).strip() if vm else 'MISSING')
else:
    print('MISSING')
"
```

Replace `<name>` with the actual skill name.

### For Plugins — Check Directory Existence then Read plugin.json

Note: Check directory existence as the primary test, NOT marketplace.json — a plugin directory may exist without a marketplace.json entry.

```bash
# Check existence
[ -d "/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/plugins/<name>" ] && echo "present" || echo "absent"

# Read version (if present)
python3 -c "
import json
d = json.load(open('/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/plugins/<name>/.claude-plugin/plugin.json'))
print(d.get('version', 'MISSING'))
"
```

Replace `<name>` with the actual plugin name.

---

## Cache Lookup

Applies to plugins only — skills do not have a Claude Code install cache entry.

```bash
python3 -c "
import json
cache = json.load(open('/Users/michaeleast/.claude/plugins/installed_plugins.json'))
plugins = cache.get('plugins', {})
key = '<plugin-name>@mikey-skills'
entry = plugins.get(key)
if entry:
    print(entry[0].get('version', 'MISSING'))
else:
    print('NOT_INSTALLED')
"
```

Replace `<plugin-name>` with the actual plugin name.

**Important:** The cache key format is `<plugin-name>@mikey-skills` — the `@mikey-skills` suffix comes from the marketplace name field. Using the wrong key format returns a false NOT_INSTALLED result even when the plugin is actually installed.

---

## Version Normalisation

Apply these rules before any version comparison to handle format differences between local and marketplace versions.

1. Strip any leading `v` or `V` prefix before comparison (e.g., `v1.2.0` becomes `1.2.0`)
2. If the version string contains no dots, treat as `N.0.0` (e.g., `2` becomes `2.0.0`, `v2` becomes `2.0.0`)
3. After normalisation, compare as tuples of integers: `tuple(int(x) for x in version.split('.'))`
4. Same normalised version = `In sync`
5. Local normalised version higher = `Local ahead`

---

## Status Label Logic

Apply in this precedence order — stop at the first matching condition:

1. Skill Version Reading returns `NO_VERSION` (SKILL.md found but no `version:` key in frontmatter) → `⚠️ Missing version`
2. Skill Version Reading returns `MISSING` (no SKILL.md at any candidate path) OR plugin Version Reading returns `MISSING` (no plugin.json) → `⚠️ Missing manifest`
3. Marketplace entry does not exist (absent ZIP for skills, absent directory for plugins) → `❌ Not published`
4. Local version equals marketplace version after normalisation → `✅ In sync`
5. Local version higher than marketplace version after normalisation → `⚠️ Local ahead`

---

## Card Format Template

### Skills Card (Local + Marketplace only — no Cache row)

```
### <name> (skill)
- Local:       skills/<name>/ [v<version>]
- Marketplace: v<version> at mikey-skills-marketplace/skills/<name>/ | not published
- Status:      <status label>
```

If local version is `MISSING`, show `[⚠️ missing manifest]` instead of `[v<version>]`. If local version is `NO_VERSION`, show `[⚠️ no version]` instead of `[v<version>]`.

For the Marketplace row:
- If published: `v<version> at mikey-skills-marketplace/skills/<name>/`
- If not published: `not published`

### Plugins Card (Local + Marketplace + Cache)

```
### <name> (plugin)
- Local:       plugins/<name>/ [v<version>]
- Marketplace: v<version> at mikey-skills-marketplace/plugins/<name>/ | not published
- Cache:       v<version> at ~/.claude/plugins/cache/mikey-skills/<name>/<version>/ | not installed
- Status:      <status label>
```

For the Marketplace row:
- If published: `v<version> at mikey-skills-marketplace/plugins/<name>/`
- If not published: `not published`

For the Cache row:
- If installed: `v<version> at ~/.claude/plugins/cache/mikey-skills/<name>/<version>/`
- If not installed: `not installed`

**Never omit a row for a missing source** — always show the row with the descriptive label ("not published", "not installed").

---

## Git Divergence Footer

Appears after all artifact cards, never inline per artifact.

### Safety Check First

Before running git commands, verify the remote exists:

```bash
git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace remote -v 2>/dev/null | grep -q origin && echo "remote_ok" || echo "no_remote"
```

If no remote exists, show this warning instead of assuming clean:

```
⚠️ Marketplace repo: remote not configured — cannot check upload status
```

### Git Commands (run only if remote exists)

```bash
# Count uncommitted changes
git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace status --porcelain 2>/dev/null | wc -l | tr -d ' '

# Count unpushed commits
git -C /Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace log origin/main..HEAD --oneline 2>/dev/null | wc -l | tr -d ' '
```

### Render Rules

- Both counts are 0: `✅ Marketplace repo: clean (all saved, all uploaded)`
- Uncommitted count greater than 0: show `⚠️ Marketplace repo needs attention:` then `- N files changed but not saved (uncommitted)`
- Unpushed count greater than 0: show `- N saves not yet uploaded to GitHub (unpushed)`

Plain-language labels are intentional — "uncommitted" and "unpushed" are shown in parentheses for those who know git, but the plain description comes first for those still learning.

---

## Summary Footer

Format: `N artifacts • N in sync • N not published`

Count each status label across all artifacts and render the three counts separated by bullet separators (` • `). "In sync" counts only `✅ In sync` labels. "Not published" counts only `❌ Not published` labels. Total artifacts is the sum of all skills and plugins discovered.

---

## Brief Mode

Brief mode is the `/marketplace-manager:status --brief` (alias `--list`) output path. It reuses the existing [Artifact Discovery](#artifact-discovery), [Version Reading](#version-reading), and [Version Normalisation](#version-normalisation) procedures to enumerate artifacts and read their local versions, then renders a flat, one-line-per-artifact inventory grouped by type. Brief mode deliberately SKIPS [Marketplace Lookup](#marketplace-lookup), [Cache Lookup](#cache-lookup), [Status Label Logic](#status-label-logic), [Card Format Template](#card-format-template), [Git Divergence Footer](#git-divergence-footer), and [Summary Footer](#summary-footer) — no remote or post-processing work is performed.

### Output Format

```
## marketplace: status (brief)

### Skills (N)
- skill-name@version
- another-skill@version

### Plugins (M)
- plugin-name@version
- another-plugin@version

Run without --brief for sync state and marketplace comparison.
```

- Top heading: `## marketplace: status (brief)` — mirrors the full-mode heading with the `(brief)` suffix to distinguish modes in terminal scrollback.
- Group headings: `### Skills (N)` and `### Plugins (M)`, where `N` / `M` are the artifact counts in each group.
- Each artifact renders as `- <name>@<version>`, where `<version>` comes from the Version Reading return contract. Sort alphabetically within each group.
- Render rules by return token:
  - If Version Reading returns `<version>` (e.g. `2`, `9`, `0.3.2`), render the line as `- <name>@<version>` using that literal string.
  - If Version Reading returns `NO_VERSION` (SKILL.md found but no `version:` field), render the line as `- <name>@no-version`. The lowercase `@no-version` suffix signals "file is fine, version field is missing" — a lower-urgency convention drift, distinct from a missing manifest.
  - If Version Reading returns `MISSING` (no SKILL.md at any candidate path), render the line as `- <name>@MISSING`. The uppercase `@MISSING` suffix signals "file not found" — a higher-urgency defect that likely indicates a layout drift or filesystem issue.
  - Do not skip or mark the artifact as error — all three states are valid brief-mode outputs that map cleanly to the full-mode Status Label Logic rules 1 and 2.
- Empty group fallback: if a group contains zero artifacts, render the heading with count `0` followed by `- (none)` on the next line, e.g. `### Plugins (0)` / `- (none)`.

### Closing Hint Line

Render the following single line after one blank line, verbatim (no trailing punctuation variations):

```
Run without --brief for sync state and marketplace comparison.
```

This string is grep-asserted by phase verification — do not paraphrase.

---

## Full Output Structure

```
## marketplace: status

[card for each artifact, sorted alphabetically by name]

---

[git divergence footer]

[summary footer]
```

Render one card per artifact. After all cards, add a horizontal rule (`---`), then the git divergence footer, then the summary footer on the last line.
