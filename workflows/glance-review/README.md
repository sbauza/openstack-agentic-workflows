# Glance Review

An ACP workflow for reviewing OpenStack Glance code and specifications.

## Skills

| Skill | Description |
|-------|-------------|
| `/glance-spec-review` | Review glance-specs proposals for completeness, technical soundness, and alignment with Glance architecture |
| `/glance-code-review` | Review Glance code changes against coding conventions, multi-tenant isolation requirements, and testing standards |

## Usage

### Ambient Code Platform (ACP)

This workflow works best when the Glance and glance-specs repositories are added to your ACP session:

- **glance** — The Glance image service source code
- **glance-specs** — Specification proposals for Glance features

If repositories are not available, the workflow will guide you to add them or you can paste code/specs inline.

### Gerrit Integration

The workflow supports two modes for interacting with Gerrit:

1. **With Gerrit MCP** (preferred): Full integration via the [official gerrit-mcp-server](https://github.com/GerritCodeReview/gerrit-mcp-server). In ACP, configure via **Integrations**. For Claude Code or Cursor, see [Configuring MCP Servers](../../README.md#configuring-mcp-servers) in the main README.
   - Access to review history and patchset details
   - Future: automated review posting

2. **Without Gerrit MCP** (fallback): Limited functionality
   - Review history check is skipped
   - Manual artifact generation for review comments

The workflow automatically detects Gerrit MCP availability at startup and adapts accordingly.

### Claude Code

Run `claude` from the workflow directory to auto-load skills, rules, and personas:

```bash
cd openstack-agentic-workflows/workflows/glance-review
claude
```

Skills are available as slash commands: `/glance-spec-review`, `/glance-code-review`. Agent personas (`glance-core`, `glance-coresec`) are loaded automatically via `CLAUDE.md`.

### Cursor

Open the repository root in Cursor. Skills are discovered via symlinks in `.agents/skills/`:

| Cursor Skill | Maps To |
|--------------|---------|
| `glance-spec-review` | `/glance-spec-review` |
| `glance-code-review` | `/glance-code-review` |

Type `/` in the agent chat to invoke a skill. Agent personas are auto-detected from `agents/`.

## What It Checks

### Spec Review

- **Structural completeness** against the Glance spec template
- **Multi-tenant isolation** considerations in the design
- **glance_store integration** — whether changes require glance_store repo coordination
- **Database migration safety** — additive-only, online migrations
- **Cross-project impact** assessment
- **Risk assessment** for high-impact patterns (security, API changes, policy changes)

### Code Review

- **Multi-tenant isolation** (blocker) — image visibility rules, access control, policy checks
- **Database migrations** (blocker) — additive-only, online migrations
- **Store driver boundaries** — pluggable drivers, clean interfaces, credential isolation
- **Testing adequacy** — unit test coverage, functional tests for image operations
- **Release notes** — whether `reno` notes are needed
- **API compatibility** — backward compatibility assessment
- **Image validation** — security properties preserved in format validation changes

## Output

Review artifacts are written to `artifacts/glance-review/`:

- `spec-{name}.md` — Spec review reports
- `code-{topic}.md` — Code review reports

## File Structure

```text
workflows/glance-review/
├── .ambient/
│   └── ambient.json       # Workflow configuration
├── .claude/
│   └── skills/
│       ├── glance-spec-review/
│       │   └── SKILL.md   # Spec review skill
│       └── glance-code-review/
│           └── SKILL.md   # Code review skill
├── AGENTS.md              # Project reference (conventions, architecture)
├── CLAUDE.md              # Pointer to AGENTS.md and rules
├── rules.md               # Behavioral rules (self-review, etc.)
└── README.md              # This file
```

## Agent Personas

This workflow uses two specialized agent personas for review:

- **glance-core** (`agents/glance-core.md`) — Database migrations, multi-tenant isolation, store driver boundaries, architectural fit
- **glance-coresec** (`agents/glance-coresec.md`) — Security review for image validation, RBAC policies, credential handling

## Testing in ACP

To test this workflow before merging changes:

1. Push your branch to GitHub
2. In ACP, select "Custom Workflow..."
3. Enter the following fields:

| Field | Value |
|-------|-------|
| **URL** | `https://github.com/sbauza/openstack-agentic-workflows.git` |
| **Branch** | Your branch name (e.g., `feature/glance-review`) |
| **Path** | `workflows/glance-review` |

## Future Enhancements

- [ ] Add `/glance-gerrit-comment` skill for posting reviews to Gerrit
- [ ] Expand `knowledge/glance.md` with Glance-specific architectural patterns
- [ ] Add functional test review guidelines
- [ ] Add store driver-specific review patterns
