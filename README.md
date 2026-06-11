# Agent Skills

Reusable Codex skills for consistent development workflows across projects.

## Available Skills

### `use-zustand-modal-pattern`

Standardizes TypeScript React modals around:

- A centralized Zustand modal store
- Typed modal payloads
- Dedicated trigger and modal components
- shadcn/ui and Radix Dialog lifecycle handling
- React Hook Form and Zod form modals
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
