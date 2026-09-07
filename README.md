# frontend-skills

Reusable frontend engineering conventions for AI coding agents, packaged as portable [Agent Skills](https://agentskills.io/).

This repository contains frontend development rules as installable Skills that can be used across compatible coding agents and projects.

## Structure

```text
frontend-skills/
│
├── develop/
│   ├── SKILL.md
│   └── references/
│       ├── GENERAL.md
│       ├── REPOSITORY.md
│       ├── SERVICE.md
│       ├── STORE.md
│       ├── COMPONENT.md
│       ├── VIEW.md
│       ├── COMPOSABLE.md
│       ├── ...
│
├── README.md
└── LICENSE
```

Each top-level directory is an independently installable Skill.

## Skills

### `develop`

Frontend development conventions covering architecture, code style, naming, components, state management, styling, localization, and other project-level development practices.

The Skill uses `SKILL.md` as its entry point and `references/` for detailed rules.

## Installation

Install the `develop` Skill into your project:

```bash
npx skills add https://github.com/eghamat24/frontend-skills --skill develop
```

To install all available Skills:

```bash
npx skills add https://github.com/eghamat24/frontend-skills
```

You can replace the repository URL with your own fork.

## Adding a Skill

Add a new top-level directory containing a `SKILL.md` and, optionally, a `references/` directory:

```text
<skill-name>/
├── SKILL.md
└── references/
```

Keep `SKILL.md` focused on the Skill's purpose, workflow, and instructions. Put detailed rules and supporting documentation in `references/`.
