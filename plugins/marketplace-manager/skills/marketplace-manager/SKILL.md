---
description: "Routes distribution requests to the correct marketplace command through natural conversation. Understands publishing, validating, scaffolding, and checking status of skills and plugins. Use when managing skills or plugins through conversation instead of slash commands. Triggers on: publish my skill, ship it, validate my plugin, what is live in my marketplace, create a new skill, check status, what needs publishing, I have finished building X, release X. Do NOT use for building skill content, editing skill logic, or coding."
allowed-tools: Read
effort: medium
---

# Marketplace Manager

Natural language interface for the marketplace plugin. Routes distribution requests to the correct command workflow.

## Intent Routing

Route each request to the correct command based on intent. Extract the artifact name from the user's message and pass it as the argument.

| Intent Category | Example Phrases | Route To | Argument |
|---|---|---|---|
| Publish | "publish X", "ship X", "release X", "push X to marketplace", "I have finished building X" | `/marketplace-manager:publish` | artifact name |
| Validate | "validate X", "check X is valid", "is X ready to publish?", "run validation on X" | `/marketplace-manager:validate` | artifact name |
| New skill | "new skill called X", "create a skill X", "scaffold a skill named X" | `/marketplace-manager:new-skill` | skill name |
| New plugin | "new plugin called X", "create a plugin X", "scaffold a plugin named X" | `/marketplace-manager:new-plugin` | plugin name |
| Status | "what is live in my marketplace?", "status", "what needs publishing?", "show me the marketplace", "check the status", "what skills have not been published yet?" | `/marketplace-manager:status` | none |

These examples are not exhaustive. Understand natural variations — if someone says "ship my plugin" instead of "publish my plugin", route to publish. Use language understanding, not exact matching.

## Before Executing Any Command

Before executing the routed command, output one narration line to confirm what you are about to do. Format: "Running the [command-description] for [name]..." For status (no name): "Checking marketplace status..." Then proceed with the command workflow — the command will show its own output (tick marks, prompts, reports).

## When No Artifact Name Is Given

If the user's request implies a specific artifact but does not name one (for example: "publish my latest changes" or "validate it"):

1. Run the `/marketplace-manager:status` command first to show what exists and what is out of date
2. Then ask: "Which of the above would you like to [action]?"

Never ask "which one?" without first showing context via the status command.

## Unrecognised Requests

If no intent matches the user's request, respond with this capability menu — never ignore silently:

I handle the distribution side of skills and plugins. Here is what I can help with:
- Publish to the marketplace: "publish recipe-optimiser"
- Validate structure and spec compliance: "validate my recipe-optimiser"
- Check marketplace status: "what is live in my marketplace?"
- Create a new skill: "new skill called my-new-skill"
- Create a new plugin: "new plugin called my-new-plugin"

## Important Rules

- This skill routes to existing commands — it NEVER re-implements command logic (no validation rules, no packaging steps, no git operations)
- The validation gate in the publish pipeline is non-negotiable — do NOT offer to skip validation even if the user asks
- Do NOT load reference files from other commands (validation-rules.md, publish-rules.md, etc.) — those commands load their own references when executed
- If the user asks about something outside distribution (writing skill content, debugging code, etc.), use the unrecognised request menu above
