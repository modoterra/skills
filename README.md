# Modoterra Skills

Agent skills maintained by [Modoterra Corporation](https://github.com/modoterra).

Licensed under the [MIT License](LICENSE).

## Install

Use the [Skills CLI](https://github.com/vercel-labs/skills) (`npx skills`) to install into Agent Skills-compatible agents:

```bash
# project-level (default)
npx skills add modoterra/skills

# user-wide (all projects)
npx skills add modoterra/skills -g

# non-interactive: all skills, all agents
npx skills add modoterra/skills --all

# list skills in this repo without installing
npx skills add modoterra/skills --list

# install a single skill by directory name
npx skills add modoterra/skills -s swe -y
```

Update installed skills later with `npx skills update`.

## Skills

| Skill | Path | Description |
|-------|------|-------------|
| **modoterra-swe** | [`skills/swe/SKILL.md`](skills/swe/SKILL.md) | Modoterra software-engineering standards for implementation, debugging, refactoring, testing, code review, dependencies, migrations, documentation, and Git workflows. |
| **compose** | [`skills/compose/SKILL.md`](skills/compose/SKILL.md) | Docker Compose stacks with collision-resistant high host ports, project-named networks, service-named volumes, and wiring those ports into app config (e.g. Laravel `.env`). |

## Layout

```
skills/
├── swe/
│   └── SKILL.md   # modoterra-swe
└── compose/
    └── SKILL.md   # compose
```

Each skill lives under `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`, `metadata`) for Agent Skills-compatible coding agents.
