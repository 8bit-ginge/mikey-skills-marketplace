<!-- generated-by: gsd-doc-writer -->
# Development

Guide for working on the `marketplace-manager` plugin itself — modifying skills, adding commands, changing reference files, and testing changes locally.

---

## Local Setup

This plugin has no package manager or install step. It is a Claude Code plugin loaded directly from the filesystem.

1. Clone the repository (or navigate to the existing directory):
   ```bash
   cd /Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/marketplace-manager
   ```

2. Load the plugin into a Claude Code session without installing it:
   ```bash
   claude --plugin-dir ./marketplace-manager
   ```
   Or with the `cc` alias:
   ```bash
   cc --plugin-dir ./marketplace-manager
   ```

3. Verify the plugin loaded by running `/help` in the session — all six `/marketplace-manager:*` commands should appear.

---

## Build Commands

This plugin has no build step. There are no compile, transpile, or bundle operations. Changes to `.md` files take effect immediately after a plugin reload.

---

## Development Workflow

### Reloading Changes

After editing any `SKILL.md` or reference file, pick up changes without restarting Claude Code:

```
/reload-plugins
```

This is the primary dev loop action. Use it after every edit.

### Validating Plugin Structure

Check that `plugin.json` schema, skill frontmatter syntax, and hooks are valid:

```bash
claude plugin validate
```

Or from inside a Claude Code session:
```
/plugin validate
```

### Checking Context Budget

Verify skill descriptions are loading and diagnose context issues:

```
/context
```

### Version Check

Confirm the minimum required Claude Code version is installed:

```bash
claude --version
```

Minimum required: Claude Code `1.0.33` (first release with plugin support).

---

## Directory Structure

```
marketplace-manager/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest — name, version, author
├── skills/
│   ├── validate/
│   │   ├── SKILL.md             # Command definition + orchestration logic
│   │   └── references/
│   │       └── validation-rules.md
│   ├── new-skill/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── scaffold-templates.md
│   ├── new-plugin/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── scaffold-templates.md
│   ├── status/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── status-rules.md
│   ├── publish/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── publish-rules.md
│   └── marketplace-manager/
│       └── SKILL.md             # Natural language router — no references dir
└── docs/                        # Project documentation
```

**Key constraint:** Only `plugin.json` lives inside `.claude-plugin/`. All other directories (`skills/`, `docs/`, etc.) live at the plugin root level.

---

## Code Style

There is no linter or formatter. The conventions below are enforced by the `validate` command itself — a non-conforming skill will fail validation before it can be published.

### Naming

- All skill directory names: kebab-case only (e.g., `new-skill`, `marketplace-manager`)
- No uppercase, no underscores, no spaces

### SKILL.md Frontmatter Rules

| Field | Constraint |
|-------|------------|
| `description` | Max 1024 characters; no `<` or `>` characters; third person; must state what, when, and negative triggers |
| `allowed-tools` | Scope to minimum required — never use bare `Bash` without a command scope |
| `context` | Leave unset — do not use `context: fork` |
| `version` | Do not add to SKILL.md frontmatter; version is tracked in `plugin.json` only |

Always verify `description` character count before committing. Truncation at 1024 chars silently breaks natural language triggering.

### SKILL.md Body Rules

- Keep the body under 500 lines
- Move detailed logic into `references/` files, loaded on demand via explicit `Read:` instructions
- Reference files: one level deep maximum (`skills/<name>/references/<file>.md`)
- Reference files over 100 lines must include a table of contents
- Load reference files with explicit named `Read:` instructions — glob-based discovery returns no results at runtime

### plugin.json

- `version` field must follow semver: `X.X.X`
- This is the single source of truth for the plugin version

---

## Adding a New Command

1. Create the skill directory: `skills/<command-name>/`
2. Create `skills/<command-name>/SKILL.md` with required YAML frontmatter
3. If the command needs reference data, create `skills/<command-name>/references/<name>.md`
4. Add a `Read: ${CLAUDE_SKILL_DIR}/references/<name>.md` instruction in the SKILL.md body to load it
5. Run `/reload-plugins` to pick up the new skill
6. Run `/marketplace-manager:validate` to confirm the new skill passes all checks

**Do not use the legacy `commands/<name>.md` format.** Skills format (`skills/<name>/SKILL.md`) is the current standard and supports the `references/` subdirectory.

---

## Modifying Reference Files

Reference files contain the runtime logic for each command — validation rules, scaffold templates, publish pipeline steps, and ecosystem paths. They are loaded by the skill at execution time, not at load time.

To modify a reference file:

1. Edit the file directly in `skills/<skill-name>/references/<name>.md`
2. Run `/reload-plugins` — no restart needed
3. Invoke the relevant command to verify the change behaves correctly

**Ecosystem paths are hardcoded in reference files**, not in environment variables. If a path needs updating, edit the reference file for each affected command. See `docs/CONFIGURATION.md` for the full path inventory.

---

## Path Environment Variables

Use these variables in SKILL.md body and reference files to construct paths:

| Variable | Resolves To |
|----------|-------------|
| `${CLAUDE_PLUGIN_ROOT}` | Absolute path to the plugin root directory |
| `${CLAUDE_SKILL_DIR}` | Absolute path to the current skill's directory |
| `${CLAUDE_PLUGIN_DATA}` | User-scoped persistent data directory for this plugin |

---

## Branch Conventions

No convention documented. This is a personal ecosystem plugin developed on a single working branch.

---

## PR Process

No PR process documented. This is a personal project — changes are committed and pushed directly.

---

## What to Avoid

| Avoid | Use Instead |
|-------|-------------|
| `commands/<name>.md` for new commands | `skills/<name>/SKILL.md` |
| `skills` field in `plugin.json` | Leave unset — skills are auto-discovered |
| `context: fork` in any SKILL.md | Leave `context` field unset |
| Bare `allowed-tools: Bash` | Scoped form: `Bash(git *)`, `Bash(zip *)`, etc. |
| XML angle brackets in any `description` | Plain text equivalents |
| Nested reference files | One level deep only, all referenced directly from SKILL.md |
| `version` field in SKILL.md frontmatter | Track version in `plugin.json` only |
| `README.md` inside `skills/<name>/` | Document in SKILL.md body or `references/` files |
