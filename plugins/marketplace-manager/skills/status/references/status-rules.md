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
11. [Full Output Structure](#full-output-structure)

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

Read version from `<skills-dir>/<name>/claude-code-skill/SKILL.md` using frontmatter delimiter isolation. This prevents false positives from body content containing YAML-like keys.

```bash
python3 -c "
import re, sys
content = open('/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/<name>/claude-code-skill/SKILL.md').read()
m = re.search(r'^---\n(.*?)^---', content, re.MULTILINE | re.DOTALL)
if m:
    vm = re.search(r'^version:\s*(.+)$', m.group(1), re.MULTILINE)
    print(vm.group(1).strip() if vm else 'MISSING')
else:
    print('MISSING')
"
```

Replace `<name>` with the actual skill name. If the file does not exist, treat as `MISSING`.

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

1. Skill with no `version` field in frontmatter → `⚠️ Missing version`
2. No `SKILL.md` (for skills) or no `plugin.json` (for plugins) → `⚠️ Missing manifest`
3. Marketplace entry does not exist (absent ZIP for skills, absent directory for plugins) → `❌ Not published`
4. Local version equals marketplace version after normalisation → `✅ In sync`
5. Local version higher than marketplace version after normalisation → `⚠️ Local ahead`

---

## Card Format Template

### Skills Card (Local + Marketplace only — no Cache row)

```
### <name> (skill)
- Local:       skills/<name>/claude-code-skill/ [v<version>]
- Marketplace: v<version> at mikey-skills-marketplace/skills/<name>/ | not published
- Status:      <status label>
```

If local version is MISSING, show `[⚠️ no version]` instead of `[v<version>]`.

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

## Full Output Structure

```
## marketplace: status

[card for each artifact, sorted alphabetically by name]

---

[git divergence footer]

[summary footer]
```

Render one card per artifact. After all cards, add a horizontal rule (`---`), then the git divergence footer, then the summary footer on the last line.
