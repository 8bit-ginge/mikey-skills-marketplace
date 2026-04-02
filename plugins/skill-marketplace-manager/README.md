# Skill Marketplace Manager

A Claude Code plugin that handles the distribution and maintenance layer of the Claude skill/plugin lifecycle. Everything after a skill or plugin is built: validating, scaffolding, packaging, versioning, and publishing to the GitHub marketplace.

## Commands

| Command | Description |
|---------|-------------|
| `/marketplace:validate [name]` | Validate a skill or plugin against Anthropic spec and project conventions |
| `/marketplace:new-skill [name]` | Scaffold a new skill container with correct structure |
| `/marketplace:new-plugin [name]` | Scaffold a new plugin container with correct structure |
| `/marketplace:status` | Show sync state between local artifacts and the marketplace repo |
| `/marketplace:publish [name]` | Publish a skill or plugin to the GitHub marketplace |
| `/marketplace:marketplace-manager` | Natural language interface — routes conversational requests to the commands above |

## Installation

Load as a local plugin during development:

```bash
claude --plugin-dir /path/to/skill-marketplace-manager
```

Or install from the marketplace:

```bash
claude plugin add skill-marketplace-manager
```

## Core Principle

Nothing ships broken — validation is a hard gate before every package and publish operation.
