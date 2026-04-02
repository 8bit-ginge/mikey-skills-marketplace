<!-- GSD:project-start source:PROJECT.md -->
## Project

**Skill Marketplace Manager**

A Claude Code plugin that owns the **distribution and maintenance layer** of the Claude skill/plugin lifecycle. It handles everything after a skill or plugin is built: validating against Anthropic spec and project conventions, scaffolding consistent container structures, packaging for both platforms, managing versioning, and publishing to the GitHub marketplace. Designed for Mikey's personal ecosystem with a secondary natural language interface for Cowork.

**Core Value:** Nothing ships broken — validation is a hard gate before every package and publish operation.

### Constraints

- **Conventions:** Must follow `claude-ecosystem/CLAUDE.md` exactly — kebab-case naming, SKILL.md body under 500 lines, description under 1024 chars, no XML angle brackets
- **Plugin format:** `.claude-plugin/plugin.json` required for all plugins; `${CLAUDE_PLUGIN_ROOT}` for intra-plugin paths
- **Validation gate:** Non-negotiable — every publish operation must call validate first; never package with known failures
- **Git operations:** Direct Bash only — no MCP; push failures do not roll back local changes
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Executive Summary
## Recommended Stack
### Core Plugin Format
| Technology | Format | Purpose | Why |
|------------|--------|---------|-----|
| `plugin.json` | JSON, inside `.claude-plugin/` | Plugin manifest — name, version, description, author | Required entry point for the Claude Code plugin system. Without it, Claude Code does not recognise the directory as a plugin. |
| `skills/<name>/SKILL.md` | YAML frontmatter + Markdown | Each command the plugin exposes | Current standard format. Supports `references/`, `scripts/`, `examples/` subdirectories. Auto-discovered by Claude Code from the `skills/` directory. |
| `marketplace.json` | JSON, inside `.claude-plugin/` | Marketplace index — registry of plugins in a distribution repo | Required for a Git-based marketplace. Defines the `plugins` array that Claude Code reads when adding a marketplace. |
### Command Authoring
### Natural Language Skill
## File Format Specifications
### plugin.json
- Only `plugin.json` lives inside `.claude-plugin/`. Do NOT put `skills/`, `agents/`, or `hooks/` inside `.claude-plugin/`.
- All other directories (`skills/`, `documentation/`, etc.) live at the plugin root level.
- The `version` field must follow semver: `X.X.X`.
### SKILL.md Frontmatter
| Field | Required | Constraint | Usage for This Plugin |
|-------|----------|------------|-----------------------|
| `name` | No (uses dir name) | Max 64 chars; lowercase, numbers, hyphens only | Omit — directory name is sufficient |
| `description` | Recommended | Max 1024 chars; no XML angle brackets (`<` `>`); must be third person; include what + when + negative triggers | Required for all skills; critical for NL skill |
| `argument-hint` | No | Short hint shown in autocomplete, e.g., `[name]` | Use on `validate`, `new-skill`, `new-plugin`, `publish` |
| `allowed-tools` | No | Comma-separated; supports `Bash(cmd *)` scoped restrictions | Each command has a different scope — see per-command table below |
| `disable-model-invocation` | No | `true`/`false` | `false` for all — all commands are user-initiated |
| `user-invocable` | No | `false` hides from slash menu | Leave default (`true`) — all commands must appear in `/help` |
| `effort` | No | `low`, `medium`, `high`, `max` | `high` for `publish`; `medium` for `validate`, `status`, scaffolding |
| `context` | No | `fork` for subagent | Leave unset — all commands need conversation history |
| `model` | No | `haiku`, `sonnet`, `opus` | Leave unset — inherit from conversation |
| `paths` | No | Glob patterns limiting auto-activation | Leave unset |
| Command | Allowed Tools | Rationale |
|---------|---------------|-----------|
| `validate` | `Read, Glob, Bash` | Reads files, globs for existence checks, Bash for JSON parse validation |
| `new-skill` | `Write, Bash` | Creates files and directories |
| `new-plugin` | `Write, Bash` | Creates files and directories |
| `status` | `Read, Glob, Bash` | Reads files; Bash for git status queries |
| `publish` | `Read, Write, Edit, Bash, Glob` | Full pipeline: reads, writes version bumps, runs zip, runs git |
| `marketplace-manager` (NL) | `Read` | NL interface routes to command workflows; reads reference files on demand |
- Hard limit: 1024 characters including spaces
- No `<` or `>` characters anywhere in the value
- Third person ("Validates the structure..." not "I validate...")
- Must include what the skill does AND when to use it AND negative triggers
- Verify character count before committing — truncation silently breaks triggering
- Available in SKILL.md body content
- Captures everything typed after the command name
- Example: `/marketplace:validate recipe-optimiser` → `$ARGUMENTS = recipe-optimiser`
- When invoked with no arguments, `$ARGUMENTS` is empty — handle this in the skill body
### marketplace.json
### Plugin Directory Layout in Marketplace
### Commands Format (Legacy) vs Skills Format (Current)
| Format | Location | File | Status | Use When |
|--------|----------|------|--------|----------|
| **Skills** | `skills/<name>/SKILL.md` | Directory + SKILL.md | **Current recommended** | All new commands in this plugin |
| Commands | `commands/<name>.md` | Single flat `.md` file | Legacy (still works) | Never — no benefit over skills format |
### Reference Files
- One level deep maximum: `skills/marketplace-manager/references/conventions.md` — no nested references
- Any reference file over 100 lines must have a table of contents
- Load with explicit named `Read:` instructions, not glob-based discovery (glob returns no results at runtime — confirmed in skill-forge development)
- Use `${CLAUDE_PLUGIN_ROOT}` to construct paths from the plugin root
- Use `${CLAUDE_SKILL_DIR}` to construct paths relative to the current skill's directory
## What NOT to Use
| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `commands/<name>.md` for new commands | Labelled "legacy" in official docs; cannot support `references/` subdirectory | `skills/<name>/SKILL.md` |
| `skills` field in `plugin.json` pointing to `./skills/` | Redundant — auto-discovered; specifying it replaces the default, requiring explicit listing of every skills directory | Leave `skills` field unset |
| `context: fork` in any command frontmatter | Forked subagents start without conversation history; cannot access user-provided arguments or prior context | Leave `context` field unset (inline execution) |
| Unrestricted `allowed-tools: Bash` | Grants broad filesystem access unnecessarily | `Bash(git *)`, `Bash(zip *)` etc. — scope to exactly what is needed |
| XML angle brackets in any `description` field | Hard validation failure; skill will not load | Plain text equivalents |
| Deeply nested reference files | Claude partially reads files encountered from other referenced files (uses `head -100`), losing content | One level deep only; all references directly from SKILL.md |
| Embedding all command logic in SKILL.md body | Exceeds 500-line recommended limit; loads unnecessary context on every invocation | Phase/command guidance in `references/` files, loaded on demand |
| README.md inside `skills/<name>/` | Convention violation per `CLAUDE.md`; conflicts with Anthropic conventions | Documentation in SKILL.md body or `references/` |
| `version` in SKILL.md frontmatter for skills being used as commands | Version field is not part of the command frontmatter spec; only matters for standalone distributable skills | Track version in `plugin.json` only |
## Development Tooling
| Tool | Command | Purpose |
|------|---------|---------|
| Local plugin loading | `claude --plugin-dir ./skill-marketplace-manager` or `cc --plugin-dir ./skill-marketplace-manager` | Load plugin without installing; use during development |
| Reload without restart | `/reload-plugins` | Pick up SKILL.md and reference file changes without restarting Claude Code |
| Validate plugin structure | `claude plugin validate` or `/plugin validate` | Check plugin.json schema, skill frontmatter syntax, hooks.json |
| Context debugging | `/context` | Verify skill descriptions are loading; diagnose context budget issues |
| Version check | `claude --version` | Confirm Claude Code 1.0.33+ (minimum for plugin support) |
## Packaging for Marketplace Distribution
# Run from claude-code-skill/ directory
# Files/dirs to EXCLUDE from marketplace copy:
## Path Environment Variables
| Variable | Resolves To | Use When |
|----------|-------------|----------|
| `${CLAUDE_PLUGIN_ROOT}` | Absolute path to the plugin root directory | Constructing absolute paths to reference files in hooks, MCP configs, and skill body instructions |
| `${CLAUDE_SKILL_DIR}` | Absolute path to the current skill's directory | Constructing paths to the current skill's own `references/` files |
| `${CLAUDE_PLUGIN_DATA}` | User-scoped persistent data directory for this plugin | Storing plugin state across sessions (not needed for v1) |
## Alternatives Considered
| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Command format | `skills/<name>/SKILL.md` | `commands/<name>.md` | Legacy; no subdirectory support; no benefit |
| Marketplace storage | Directory copy to Git repo | ZIP archive | Claude Code marketplace installer expects raw directories, not archives |
| Version tracking | `plugin.json` version field | SKILL.md frontmatter `version` | Frontmatter version is for skills only, not command-mode SKILL.md files; single source of truth in plugin.json |
| Reference loading | Explicit `Read:` instructions in SKILL.md | Glob-based discovery | Glob returns no results at runtime (confirmed from skill-forge Phase 7 fix) |
## Sources
- `/Users/mikey-east/.claude/plugins/marketplaces/claude-plugins-official/plugins/plugin-dev/commands/create-plugin.md` — Anthropic's official plugin creation reference (HIGH confidence, read directly from installed cache 2026-03-29)
- `/Users/mikey-east/.claude/plugins/marketplaces/claude-plugins-official/plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md` — Complete frontmatter field specification (HIGH confidence, read directly 2026-03-29)
- `/Users/mikey-east/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json` — Official marketplace.json schema (HIGH confidence, read directly 2026-03-29)
- `/Users/mikey-east/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/.claude-plugin/marketplace.json` — Live mikey-skills marketplace.json (HIGH confidence, read directly 2026-03-29)
- `/Users/mikey-east/.claude/plugins/installed_plugins.json` — Confirmed plugin installation schema, cache paths, and version tracking (HIGH confidence, read directly 2026-03-29)
- `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-plugins/skill-forge/.planning/research/STACK.md` — Prior research for sibling plugin in same ecosystem, cites `https://code.claude.com/docs/en/plugins-reference`, `https://code.claude.com/docs/en/skills`, `https://code.claude.com/docs/en/plugins` (HIGH confidence, same environment)
- `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/claude-plugins/skill-forge/.claude-plugin/plugin.json` — Working production plugin.json example (HIGH confidence)
- `/Users/mikey-east/Documents/claude-code-development/resources/utilities/claude-ecosystem/CLAUDE.md` — Project conventions governing naming, packaging, and structure (HIGH confidence)
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
