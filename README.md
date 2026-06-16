# Agent Skills

Reusable Codex skills for consistent development workflows across projects.

This repository also contains reusable `AGENTS.md` templates for configuring
project-level agent behavior.

## Available Skills

### `use-zustand-modal-pattern`

Standardizes TypeScript React modals around:

- A centralized Zustand modal store
- Typed modal payloads
- Dedicated trigger and modal components
- shadcn/ui and Radix Dialog lifecycle handling
- React Hook Form and Zod form modals
- Multi-step card-selection and form modal flows
- Submission, error, reset, and accessibility behavior

[View the skill](skills/use-zustand-modal-pattern/SKILL.md)

## Install With Codex

Ask Codex:

```text
Install the use-zustand-modal-pattern skill from
https://github.com/Thangnguyenvn98/Agent-Skills/tree/main/skills/use-zustand-modal-pattern
```

Or invoke the skill installer explicitly:

```text
Use $skill-installer to install
https://github.com/Thangnguyenvn98/Agent-Skills/tree/main/skills/use-zustand-modal-pattern
```

Start a new Codex thread after installation so the skill is discovered.

## Repository Layout

```text
skills/
  skill-name/
    SKILL.md
    agents/
      openai.yaml
    references/
```

Each skill is self-contained. Add future skills as another directory under
`skills/`.

## Use In A Project

Invoke the installed skill directly:

```text
Use $use-zustand-modal-pattern when adding this TypeScript modal.
```

The skill can also trigger automatically when a task matches its description.

## Project Agent Templates

### TanStack Start

[View the TanStack Start template](templates/AGENTS.tanstack-start.md)

To configure another TanStack Start project, copy the template into that
project root as `AGENTS.md`:

```bash
curl -L \
  https://raw.githubusercontent.com/Thangnguyenvn98/Agent-Skills/main/templates/AGENTS.tanstack-start.md \
  -o AGENTS.md
```

Review the copied file and adjust its stack assumptions to match the project,
especially optional tools such as MSW, ShadCN, Zustand, React Hook Form, and
the globally installed TanStack CLI.

### TanStack Query Pagination

[View the TanStack Query pagination template](templates/AGENTS.tanstack-query-pagination.md)

Use this template when you want agents to implement page-number pagination,
load-more lists, or infinite scrolling with TanStack Query v5:

```bash
curl -L \
  https://raw.githubusercontent.com/Thangnguyenvn98/Agent-Skills/main/templates/AGENTS.tanstack-query-pagination.md \
  -o AGENTS.tanstack-query-pagination.md
```

You can copy the relevant sections into a project's `AGENTS.md` or keep it as
a companion agent instruction file.
