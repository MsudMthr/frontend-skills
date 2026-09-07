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
├── review/
│   ├── SKILL.md
│   └── references/
│       └── RISK.md
│
├── ship/
│   └── SKILL.md
│
├── vendor/
│   └── ponytail/   (git submodule — https://github.com/DietrichGebert/ponytail)
│
├── README.md
└── LICENSE
```

Each top-level directory is an independently installable Skill.

## Skills

### `develop`

Frontend development conventions covering architecture, code style, naming, components, state management, styling, localization, and other project-level development practices.

The Skill uses `SKILL.md` as its entry point and `references/` for detailed rules.

### `review`

Simplicity/YAGNI-focused code review and planning, composed from two sources:
[`vendor/ponytail`](https://github.com/DietrichGebert/ponytail) (general over-engineering/YAGNI
rules) and this repo's own `develop` skill (project-specific conventions). See `review/SKILL.md`
and `review/references/RISK.md` for the operational low/medium/high risk criteria every finding is
scored against.

### `ship`

Automates the implement → review → fix loop for a task, unit by unit: implements per `develop`'s
conventions, reviews via `/review`, auto-applies low-risk fixes, and always stops to confirm before
applying a medium/high-risk fix. See `ship/SKILL.md`.

## Submodules

This repo uses a git submodule (`vendor/ponytail`) for the `review` skill. Clone with:

```bash
git clone --recurse-submodules https://github.com/eghamat24/frontend-skills
```

If you already cloned without that flag, run:

```bash
git submodule update --init --recursive
```

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
