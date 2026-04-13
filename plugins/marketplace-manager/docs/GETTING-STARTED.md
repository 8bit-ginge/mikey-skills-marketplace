<!-- generated-by: gsd-doc-writer -->
# Getting Started

Everything you need to load the `marketplace-manager` plugin and run your first command.

---

## Prerequisites

This plugin runs entirely inside Claude Code. There is no separate runtime or package manager required.

| Requirement | Minimum Version | How to Check |
|-------------|----------------|--------------|
| Claude Code | `1.0.33` | `claude --version` |

The plugin also expects a local ecosystem layout at a specific path. See [Ecosystem Paths](#ecosystem-paths) below.

---

## Installation Steps

### Option 1 — Load locally during development (recommended)

This loads the plugin directly from the source directory without installing it permanently. Think of it like test-driving the plugin before committing to it.

```bash
claude --plugin-dir /Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/marketplace-manager
```

Or, if your working directory is already the plugin root:

```bash
claude --plugin-dir .
```

### Option 2 — Install from the marketplace

```bash
claude plugin add marketplace-manager
```

This installs from the `mikey-skills-marketplace` GitHub marketplace. Use Option 1 during active development of the plugin itself.

---

## First Run

Once the plugin is loaded, confirm it is active:

```
/marketplace-manager:status
```

This runs the status command, which reads the local ecosystem and reports the sync state of every skill and plugin. If the plugin loaded correctly, you will see a structured report listing artifacts with their local, marketplace, and install cache versions.

If the command does not appear in autocomplete, run `/reload-plugins` to pick up changes without restarting Claude Code.

---

## Ecosystem Paths

The plugin operates on a fixed local directory layout. These paths must exist on your machine for the commands to work. They are hardcoded in each command's reference file — not read from environment variables.

| Location | Path |
|----------|------|
| Skills source | `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/skills/` |
| Plugins source | `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/plugins/` |
| Marketplace repo (local) | `/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/` |
| Claude Code install cache | `~/.claude/plugins/installed_plugins.json` |

If any of these paths have moved, update the relevant reference files. See [CONFIGURATION.md](CONFIGURATION.md) — "Changing Configuration" — for which file to edit.

---

## Common Setup Issues

**Commands do not appear after loading the plugin**

Run `/reload-plugins` inside Claude Code. Changes to SKILL.md files and reference files are not picked up until you reload or restart.

**`/marketplace-manager:status` returns no artifacts**

The skills and plugins source directories are empty or the paths have changed. Verify that `skills/` and `plugins/` exist at the ecosystem root path shown above.

**`/marketplace-manager:publish` stops at the validation gate**

This is expected behaviour — nothing ships broken. Run `/marketplace-manager:validate <name>` first to get a structured report showing exactly which checks failed and how to fix them. Fix the issues and re-run publish.

**`/marketplace-manager:publish` fails at the git push step**

A failed git push does not roll back local file changes (version bump, CHANGELOG entry, packaged artifact). Check the push error output, resolve it (e.g., authentication, branch protection), and re-run from the terminal using `git push` directly from the marketplace repo directory.

---

## Next Steps

- [ARCHITECTURE.md](ARCHITECTURE.md) — How the plugin is structured internally and how the publish pipeline flows end-to-end
- [CONFIGURATION.md](CONFIGURATION.md) — All configuration values: ecosystem paths, validation rules, publish settings, and how to change them
