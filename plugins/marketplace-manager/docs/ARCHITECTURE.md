<!-- generated-by: gsd-doc-writer -->
# Architecture

The `marketplace-manager` is a Claude Code plugin that owns the **distribution and maintenance layer** of the Claude skill/plugin lifecycle. It takes over where Skill Forge ends: validating built artifacts against the Anthropic spec and project conventions, scaffolding consistent container structures, versioning, packaging, and publishing to the GitHub-hosted marketplace. It does not create skill content — that boundary is intentional.

---

## Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    marketplace-manager                     │
│                                                                 │
│  ┌───────────────┐    ┌───────────────┐    ┌─────────────────┐  │
│  │   validate    │◀───│    publish    │───▶│     status      │  │
│  │  (SKILL.md +  │    │  (9-stage     │    │  (drift report) │  │
│  │  rules ref)   │    │   pipeline)   │    │                 │  │
│  └───────────────┘    └───────┬───────┘    └─────────────────┘  │
│                               │                                  │
│  ┌───────────────┐    ┌───────▼───────┐                         │
│  │   new-skill   │    │  new-plugin   │                         │
│  │  (scaffolds   │    │  (scaffolds   │                         │
│  │  skill cont.) │    │ plugin cont.) │                         │
│  └───────────────┘    └───────────────┘                         │
│                                                                 │
│  ┌─────────────────────────────────────────┐                    │
│  │          marketplace-manager            │                    │
│  │    (natural language router — reads     │                    │
│  │     intent and delegates to above)      │                    │
│  └─────────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
         │                                │
         ▼                                ▼
┌─────────────────┐              ┌─────────────────────┐
│  skills/ │              │   plugins/   │
│  plugins/│              │ mikey-skills-market │
│  (local source) │              │    place/ (GitHub)  │
└─────────────────┘              └─────────────────────┘
```

---

## Data Flow

A typical end-to-end publish operation follows this path:

1. **Entry** — User runs `/marketplace-manager:publish <name>` (or routes through `marketplace-manager` in natural language).
2. **Type detection** — The skill resolves whether `<name>` is a skill (`skills/<name>/`) or a plugin (`plugins/<name>/`). Skills take precedence if both directories exist.
3. **Validation gate (Stage 1)** — `publish` reads `validation-rules.md` from the `validate` skill's references and runs all applicable checks. If any check fails, the pipeline stops immediately — no writes occur.
4. **User prompts (Stages 2–3)** — The version bump type (patch/minor/major) and a one-line changelog summary are collected before any files are written.
5. **PREFLIGHT (Stage 0)** — Four pre-write safety checks, including duplicate-version detection using the version proposed in Stage 2.
6. **Write stages (4–8)** — Version is bumped in `plugin.json` (plugins) or SKILL.md frontmatter (skills), CHANGELOG entry is written, the artifact is packaged, copied to the marketplace repo, the marketplace `README.md` index is regenerated, and a `git commit && push` is run.
7. **Output** — Each stage emits a visible progress line. In `--dry-run` mode every line is prefixed `[DRY RUN]` and no filesystem writes occur.

---

## Key Abstractions

| Abstraction | File | Purpose |
|-------------|------|---------|
| `validation-rules.md` | `skills/validate/references/validation-rules.md` | Single source of truth for all check definitions, pass/fail criteria, fix message templates, and ecosystem paths used by `validate` and `publish` |
| `publish-rules.md` | `skills/publish/references/publish-rules.md` | Full pipeline specification: stage definitions, packaging commands, output format contracts, dry-run rules, and failure handling |
| `scaffold-templates.md` | `skills/new-skill/references/scaffold-templates.md` and `skills/new-plugin/references/scaffold-templates.md` | Container templates, guard check logic, and directory creation order for scaffolding commands |
| `status-rules.md` | `skills/status/references/status-rules.md` | Ecosystem paths, artifact discovery commands, version-reading procedures, status label logic, and card format templates |
| `plugin.json` | `.claude-plugin/plugin.json` | Plugin manifest — name, version, description, author. Version `0.2.0` is the single source of truth for the plugin's own release version |
| Validation gate | Stage 1 of `publish-rules.md` | Hard stop before any writes — re-uses `validate` logic inline; cannot be bypassed |
| Dry-run mode (`--dry-run`) | `publish-rules.md` D-12 contract | Full pipeline simulation with `[DRY RUN]` prefix on every output line; write stages print what they would do without touching the filesystem |

### Design principle: thin SKILL.md, fat references

Each `SKILL.md` is a thin orchestrator. All execution details live in a `references/` file loaded via an explicit `Read:` instruction at the top of the skill body. This keeps SKILL.md bodies well under the 500-line limit and avoids loading unnecessary context on every invocation.

---

## Directory Structure

```
marketplace-manager/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest (name, version, author)
├── skills/
│   ├── validate/
│   │   ├── SKILL.md             # Thin orchestrator; loads validation-rules.md
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
│   │   ├── SKILL.md             # Thin orchestrator; loads publish-rules.md
│   │   └── references/
│   │       └── publish-rules.md
│   └── marketplace-manager/
│       └── SKILL.md             # NL router — no references subdir
├── documentation/
│   └── development-plan.md
├── docs/                        # Generated documentation
├── README.md
└── CHANGELOG.md
```

**Rationale by directory:**

- `.claude-plugin/` — Required by the Claude Code plugin system. Only `plugin.json` lives here; skills, commands, and hooks live at the plugin root per the spec.
- `skills/` — Auto-discovered by Claude Code. Each subdirectory is one command the plugin exposes. Skills format is used throughout (no legacy `commands/` flat files).
- `skills/<name>/references/` — One level of reference files only. All execution logic lives here, not in SKILL.md bodies. Loaded via explicit `Read:` instructions (glob-based discovery does not work at runtime).
- `documentation/` — Dev plans and research notes. Not shipped to the marketplace (excluded during packaging).

---

## Ecosystem Paths

The plugin operates on a shared local ecosystem. These paths are the ground truth for locating artifacts — defined once in each reference file, never embedded in SKILL.md bodies:

| Location | Path |
|----------|------|
| Ecosystem root | `/Users/michaeleast/Documents/claude-code-development/resources/utilities/claude-ecosystem/` |
| Skills source | `claude-ecosystem/skills/` |
| Plugins source | `claude-ecosystem/plugins/` |
| Marketplace repo (local) | `/Users/michaeleast/Documents/claude-code-development/resources/utilities/mikey-skills-marketplace/` |
| Marketplace remote | `https://github.com/8bit-ginge/mikey-skills-marketplace.git` |
| Claude Code install cache | `~/.claude/plugins/installed_plugins.json` |

---

## Architectural Boundaries

**This plugin does not:**
- Create skill or plugin *content* — that is Skill Forge's responsibility
- Use MCP servers — all git operations run via scoped `Bash` tool calls
- Run hooks — no `hooks.json`; all operations are user-initiated
- Perform ecosystem-wide publish in a single command — `publish` always requires an explicit artifact name

**Validation is a hard gate, not advisory.** The `publish` command runs the full validation check set before any file write. There is no flag to skip or override validation. If checks fail, the pipeline stops and outputs the full validation report with fix guidance.
