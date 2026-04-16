<!-- generated-by: gsd-doc-writer -->
# Marketplace Manager

A Claude Code plugin that owns the distribution and maintenance layer of the Claude skill/plugin lifecycle. Handles everything after a skill or plugin is built: validating against Anthropic spec and project conventions, scaffolding consistent container structures, managing versioning, and publishing to the GitHub marketplace.

**Core principle:** Nothing ships broken — validation is a hard gate before every package and publish operation.

## Commands

| Command | Description |
|---------|-------------|
| `/marketplace-manager:validate [name]` | Validate a skill or plugin against Anthropic spec and project conventions. Produces a structured pass/fail report with fix guidance. |
| `/marketplace-manager:new-skill [name]` | Scaffold a new skill container with all required files that pass validation immediately. |
| `/marketplace-manager:new-plugin [name]` | Scaffold a new plugin container with all required files that pass validation immediately. |
| `/marketplace-manager:status` | Show sync state of every skill and plugin across local, marketplace repo, and install cache. |
| `/marketplace-manager:publish [name] [--dry-run]` | Run the full publish pipeline: validate, version bump, changelog, package, copy to marketplace, regenerate index, git commit and push. |
| `/marketplace-manager:marketplace-manager` | Natural language interface — routes conversational requests to the correct command above. |

## Installation

Load as a local plugin during development:

```bash
claude --plugin-dir /path/to/marketplace-manager
```

Or install from the marketplace:

```bash
claude plugin add marketplace-manager
```

## Quick Start

1. Load the plugin with `claude --plugin-dir ./marketplace-manager`
2. Validate an existing skill or plugin before publishing:
   ```
   /marketplace-manager:validate my-skill-name
   ```
3. If validation passes, publish to the marketplace:
   ```
   /marketplace-manager:publish my-skill-name
   ```
4. Use `--dry-run` to rehearse the publish pipeline without making changes:
   ```
   /marketplace-manager:publish my-skill-name --dry-run
   ```

## Usage Examples

**Check what is live in the marketplace:**
```
/marketplace-manager:status
```

**Scaffold a new skill from scratch:**
```
/marketplace-manager:new-skill recipe-optimiser
```
This creates `skills/recipe-optimiser/SKILL.md` and all required supporting files, pre-validated.

**Use natural language instead of slash commands:**
```
/marketplace-manager:marketplace-manager
```
Then say things like "I've finished building my new skill, ship it" or "validate everything" and the manager routes the request to the right command.

## Version

Current version: `0.3.1` — see [CHANGELOG.md](CHANGELOG.md) for release history.

## License

Personal use. Part of Mikey East's Claude ecosystem.
