---
name: status
description: "Shows the sync state of every skill and plugin across local development, the marketplace repo, and the Claude Code install cache. Outputs a detailed card per artifact with version, source paths, and status label. Pass --brief (alias --list) for a one-line-per-artifact inventory that skips marketplace and cache lookups. Use when you want to see what is published, what is ahead of the marketplace, or what has never been published. Triggers on: status, what is published, show marketplace state, check sync, what needs publishing, inventory, list plugins. Do NOT use for validating or publishing."
argument-hint: "[--brief | --list]"
allowed-tools: Read, Glob, Bash
effort: medium
---

# Marketplace: Status

Read: ${CLAUDE_SKILL_DIR}/references/status-rules.md

## How This Command Works

This command shows the sync state of every skill and plugin in the ecosystem. All lookup procedures, ecosystem paths, card format templates, status label definitions, and git commands are in the reference file loaded above.

## Procedure

0. Parse `$ARGUMENTS` to detect the mode flag:
   - Lowercase the argument for comparison.
   - If the lowercased argument equals `--brief` or `--list`, follow the "Brief Mode" procedure below (Brief Mode subsection) and do NOT execute steps 1-6.
   - `--list` is an alias that normalises to `--brief` — the two flags produce identical output.
   - Any other argument value (including empty `$ARGUMENTS`) falls through to the full-mode procedure in steps 1-6 unchanged.

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

## Brief Mode

Invoked when `$ARGUMENTS` matches `--brief` or `--list` (case-insensitive) per step 0 of the Procedure. See the "Brief Mode" section of status-rules.md for the output format template, empty-group fallback, and the exact closing hint line.

Brief Mode procedure:

1. Read the "Artifact Discovery" section from status-rules.md and enumerate all skills and plugins in the ecosystem (same Bash commands as full mode — reuse, do not duplicate).
2. For each discovered artifact, read its local version using the "Version Reading" procedure from status-rules.md — YAML frontmatter isolation for skills, `plugin.json` parse for plugins. If the procedure returns `MISSING`, keep the literal string `MISSING` for rendering.
3. Sort skills alphabetically; sort plugins alphabetically (same ordering rule as full mode).
4. Render the output using the "Brief Mode" format template from status-rules.md:
   - Top heading `## marketplace: status (brief)`
   - `### Skills (N)` heading followed by one `- <name>@<version>` line per skill (or `- (none)` if N = 0)
   - `### Plugins (M)` heading followed by one `- <name>@<version>` line per plugin (or `- (none)` if M = 0)
   - One blank line, then the verbatim closing hint line from status-rules.md
5. Do NOT perform Marketplace Lookup, Cache Lookup, Status Label evaluation, Card Format rendering, Git Divergence check, or Summary Footer rendering — brief mode skips all of these.

## Important Rules

- All ecosystem paths come from status-rules.md — do NOT hardcode paths in this file
- Load status-rules.md via the Read instruction above — do NOT use Glob to find it
- Skills show Local + Marketplace rows only (no Cache row) — per source mapping rules in status-rules.md
- Plugins show Local + Marketplace + Cache rows
- Missing sources show descriptive labels ("not published", "not installed") — never omit the row
- Use YAML frontmatter delimiter isolation for version reading — NEVER regex the full SKILL.md file
- Sort artifacts alphabetically by name
- Git divergence footer appears AFTER all artifact cards, never inline per artifact
- When `$ARGUMENTS` is `--brief` or `--list` (case-insensitive), follow the Brief Mode procedure and skip all marketplace, cache, and git divergence lookups
- Brief Mode reuses the existing Artifact Discovery and Version Reading procedures from status-rules.md — do NOT duplicate enumeration or version-read logic
