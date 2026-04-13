<!-- generated-by: gsd-doc-writer -->
# Configuration

Reference for all configuration values that control how the `marketplace-manager` plugin locates, validates, and publishes skills and plugins.

---

## Ecosystem Paths

The plugin does not use environment variables or a config file. All paths are hardcoded in the reference files loaded by each command at runtime. These paths are the ground truth for every command in the plugin.

| Setting | Value | Used By |
|---------|-------|---------|
| Ecosystem root | `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/` | All commands |
| Skills directory | `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/` | `validate`, `new-skill`, `status`, `publish` |
| Plugins directory | `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/` | `validate`, `new-plugin`, `status`, `publish` |
| Marketplace repo | `/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/` | `status`, `publish` |
| Claude Code install cache | `~/.claude/plugins/installed_plugins.json` | `status` |

These paths are not read from environment variables. They are defined in each command's reference file. To change a path, update the relevant reference file directly.

**Where each path lives:**

| Reference File | Commands That Load It |
|---------------|----------------------|
| `skills/validate/references/validation-rules.md` | `validate`, `publish` (Stage 1 loads this file) |
| `skills/publish/references/publish-rules.md` | `publish` |
| `skills/status/references/status-rules.md` | `status` |
| `skills/new-skill/references/scaffold-templates.md` | `new-skill` |
| `skills/new-plugin/references/scaffold-templates.md` | `new-plugin` |

---

## Plugin Manifest

The plugin manifest lives at `.claude-plugin/plugin.json` in the plugin root. This file is required for Claude Code to recognise the directory as a plugin.

**Location:** `.claude-plugin/plugin.json`

```json
{
  "name": "marketplace-manager",
  "description": "Distribution and maintenance plugin for Claude skills and plugins — validate, scaffold, package, and publish to the GitHub marketplace.",
  "version": "0.2.0",
  "author": {
    "name": "Mikey East"
  }
}
```

**Fields:**

| Field | Required | Constraint | Description |
|-------|----------|------------|-------------|
| `name` | Yes | Kebab-case; matches directory name | Plugin identifier used by Claude Code |
| `description` | Yes | Max 1024 chars; no `<` or `>` | Shown in Claude Code plugin list |
| `version` | Yes | Semver `X.X.X` | Single source of truth for plugin version |
| `author.name` | Yes | String | Plugin author |

The `version` field is the canonical version for the entire plugin. It is the value bumped during `/marketplace-manager:publish` and read by `/marketplace-manager:status` when reporting plugin sync state.

---

## Skill Frontmatter

Each skill is configured via YAML frontmatter at the top of its `SKILL.md` file. This controls how Claude Code discovers, presents, and invokes the skill.

**Location:** `skills/<name>/SKILL.md`

**Frontmatter fields in use across this plugin's skills:**

| Field | Required | Constraint | Skills That Use It |
|-------|----------|------------|--------------------|
| `name` | Recommended | Max 64 chars; lowercase, numbers, hyphens only | All except `marketplace-manager` |
| `description` | Yes | Max 1024 chars; no `<` or `>`; third person | All |
| `argument-hint` | No | Short hint shown in autocomplete | `validate`, `new-skill`, `new-plugin`, `publish` |
| `allowed-tools` | No | Comma-separated tool names | All |
| `effort` | No | `low`, `medium`, `high`, `max` | All |

**Per-skill tool permissions:**

| Skill | `allowed-tools` | Rationale |
|-------|----------------|-----------|
| `validate` | `Read, Glob, Bash` | Reads files; Bash for JSON parse and directory existence checks |
| `new-skill` | `Write, Bash` | Creates files and directories |
| `new-plugin` | `Write, Bash` | Creates files and directories |
| `status` | `Read, Glob, Bash` | Reads files; Bash for git status queries |
| `publish` | `Read, Write, Edit, Bash, Glob` | Full pipeline: reads, writes version bumps, runs zip, runs git |
| `marketplace-manager` | `Read` | Natural language router; reads reference files on demand |

---

## Validation Rules

Validation behaviour is configured entirely in `skills/validate/references/validation-rules.md`. This file is the single source of truth for every check the `/marketplace-manager:validate` command performs.

**Check categories:**

| Category | Applies To | Reference Section |
|----------|------------|-------------------|
| Container structure | Skills and plugins | `Category 1: Container Structure (VAL-03)` |
| SKILL.md frontmatter | Skills only | `Category 2: SKILL.md Frontmatter (VAL-04, VAL-08)` |
| Plugin JSON | Plugins only | `Category 3: Plugin JSON (VAL-05)` |
| Documentation | Skills and plugins | `Category 4: Documentation (VAL-06)` |

**Key limits enforced by validation:**

| Constraint | Limit | Check ID |
|------------|-------|----------|
| `description` character count | 1024 chars | VAL-04 |
| SKILL.md body line count | 500 lines | VAL-04 |
| Name format | Kebab-case `^[a-z0-9][a-z0-9-]*[a-z0-9]$` | VAL-03 |
| Plugin JSON `version` format | Semver `^[0-9]+\.[0-9]+\.[0-9]+$` | VAL-05 |

To change a validation rule (threshold, check behaviour, fix message), edit `skills/validate/references/validation-rules.md` directly. The same rules file is also loaded by `/marketplace-manager:publish` at Stage 1.

---

## Publish Pipeline

The publish pipeline is configured in `skills/publish/references/publish-rules.md`. No environment variables are needed for local publishing. The marketplace Git remote is hardcoded in that reference file.

**Hardcoded publish settings:**

| Setting | Value | Location |
|---------|-------|----------|
| Marketplace Git remote | `https://github.com/8bit-ginge/mikey-skills-marketplace.git` | `publish-rules.md` — Ecosystem Paths section |
| Marketplace name (cache key) | `mikey-skills` | `status-rules.md` — Ecosystem Paths section |

**Pipeline stages:**

| Stage | Description | Writes Files |
|-------|-------------|-------------|
| 1 | Validation gate — hard stop if any checks fail | No |
| 2 | Version bump prompt — collect bump type from user | No |
| 0 | Preflight checks (runs after Stage 2) | No |
| 3 | Changelog summary prompt — collect release notes | No |
| 4 | Write version bump | Yes |
| 5 | Write CHANGELOG entry | Yes |
| 6 | Package artifact | Yes |
| 7 | Copy to marketplace and update index | Yes |
| 8 | Git commit and push | Yes |

The `--dry-run` flag runs the full pipeline in simulation mode. All output is prefixed with `[DRY RUN]` and no files are written during Stages 4–8.

---

## Path Environment Variables

These variables are available inside SKILL.md body content. They resolve at runtime and are used to construct paths without hardcoding absolute values in the skill body itself.

| Variable | Resolves To | Use Case |
|----------|-------------|----------|
| `${CLAUDE_PLUGIN_ROOT}` | Absolute path to the plugin root directory | Reference file paths in `Read:` instructions |
| `${CLAUDE_SKILL_DIR}` | Absolute path to the current skill's directory | Paths to the skill's own `references/` files |
| `${CLAUDE_PLUGIN_DATA}` | User-scoped persistent data directory | Storing plugin state across sessions (not used in v1) |

**Example from `validate/SKILL.md`:**
```
Read: ${CLAUDE_SKILL_DIR}/references/validation-rules.md
```

---

## Scaffold Guard Constraints

The `/marketplace-manager:new-skill` and `/marketplace-manager:new-plugin` commands enforce these guards before creating any files. These are not user-configurable — they are defined in `skills/new-skill/references/scaffold-templates.md` and `skills/new-plugin/references/scaffold-templates.md`.

| Guard | Check | Failure Output |
|-------|-------|----------------|
| Empty argument | `$ARGUMENTS` must not be empty | `Error: No name provided. Usage: /marketplace-manager:new-skill [name]` |
| Name format (SCAF-03) | Must match `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$` | `Error: 'NAME' is not a valid kebab-case name.` |
| Duplicate check (SCAF-04) | Target directory must not already exist | `Error: Skill 'NAME' already exists at skills/NAME/` |

No files or directories are created until all guards pass.

---

## Changing Configuration

Because there is no central config file, updates to configuration values must be made in the reference file where they are defined.

| What to change | File to edit |
|----------------|-------------|
| Ecosystem paths (skills, plugins, marketplace) | Edit in each affected reference file: `validation-rules.md`, `publish-rules.md`, `status-rules.md`, `scaffold-templates.md` |
| Validation thresholds (description length, body line limit) | `skills/validate/references/validation-rules.md` |
| Publish pipeline behaviour | `skills/publish/references/publish-rules.md` |
| Status card format | `skills/status/references/status-rules.md` |
| Scaffold templates | `skills/new-skill/references/scaffold-templates.md`, `skills/new-plugin/references/scaffold-templates.md` |
| Plugin version | `.claude-plugin/plugin.json` — `version` field (semver `X.X.X`) |
