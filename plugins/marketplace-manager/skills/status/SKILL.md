---
name: status
description: "Shows the sync state of every skill and plugin across local development, the marketplace repo, and the Claude Code install cache. Outputs a detailed card per artifact with version, source paths, and status label. Use when you want to see what is published, what is ahead of the marketplace, or what has never been published. Triggers on: status, what is published, show marketplace state, check sync, what needs publishing. Do NOT use for validating or publishing."
allowed-tools: Read, Glob, Bash
effort: medium
---

# Marketplace: Status

Read: ${CLAUDE_SKILL_DIR}/references/status-rules.md

## How This Command Works

This command shows the sync state of every skill and plugin in the ecosystem. All lookup procedures, ecosystem paths, card format templates, status label definitions, and git commands are in the reference file loaded above.

## Procedure

1. Read the "Artifact Discovery" section from status-rules.md
2. Discover all skills and plugins in the ecosystem using the listed Bash commands
3. For each artifact found:
   a. Read version from local source using the "Version Reading" procedures — YAML frontmatter for skills, plugin.json for plugins
   b. Check marketplace presence and read marketplace version using the "Marketplace Lookup" procedures
   c. For plugins only: check install cache using the "Cache Lookup" procedure
   d. Normalise versions using the "Version Normalisation" rules
   e. Determine status label using the "Status Label Logic" precedence order
   f. Render the card using the "Card Format Template"
4. After all cards, render a horizontal rule separator (`---`)
5. Run the git divergence check using the "Git Divergence Footer" section
6. Render the summary footer using the "Summary Footer" section

## Output Structure

```
## marketplace: status

[one card per artifact -- see Card Format Template in status-rules.md]

---

[git divergence line -- see Git Divergence Footer in status-rules.md]

[summary line -- see Summary Footer in status-rules.md]
```

## Important Rules

- All ecosystem paths come from status-rules.md — do NOT hardcode paths in this file
- Load status-rules.md via the Read instruction above — do NOT use Glob to find it
- Skills show Local + Marketplace rows only (no Cache row) — per source mapping rules in status-rules.md
- Plugins show Local + Marketplace + Cache rows
- Missing sources show descriptive labels ("not published", "not installed") — never omit the row
- Use YAML frontmatter delimiter isolation for version reading — NEVER regex the full SKILL.md file
- Sort artifacts alphabetically by name
- Git divergence footer appears AFTER all artifact cards, never inline per artifact
