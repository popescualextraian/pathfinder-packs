# pathfinder-packs

A repository of Claude Code agents and skills organized by **organizational role**. Install only the area(s) relevant to your role and get the agents/skills tailored to that work.

## Layout

```
pathfinder-packs/
├── concepts/           # Concept notes describing what we do here
├── common/             # Shared agents & skills used across all roles
│   ├── agents/
│   └── skills/
├── business-analyst/   # Business Analyst role
│   ├── agents/
│   └── skills/
└── architect/          # Software Architect role
    ├── agents/
    └── skills/
```

Each top-level area (other than `common/`) represents a single role. Add a new role by creating a sibling directory with the same `agents/` + `skills/` substructure.

## Authoring conventions

- **Agents** live in `<area>/agents/<agent-name>.md` and follow the Claude Code subagent format (frontmatter with `name`, `description`, `tools`, optional `model`; body is the system prompt).
- **Skills** live in `<area>/skills/<skill-name>/SKILL.md` (plus any supporting files in the same folder) and follow the Claude Code skill format (frontmatter with `name` and `description`; body is the instructions).
- **Common** holds anything reusable across roles (e.g. document formatting, diagramming, repo navigation). Role areas should depend on `common/` rather than duplicating it.
- One concern per skill/agent. Keep descriptions specific so Claude picks the right one.

## Adding a new role

1. Create `<role-name>/agents/` and `<role-name>/skills/`.
2. Add at least one skill or agent.
3. Update this file's layout section.
