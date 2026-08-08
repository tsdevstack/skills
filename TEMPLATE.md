# Skill template

Copy this into `skills/<name>/SKILL.md`. This file is `.md`, not `SKILL.md`, so
`npx skills` will not install it as a skill.

```markdown
---
name: my-skill
description: Use when <trigger 1>, <trigger 2>. <One line on what it does.>
---

# <Title>

## When to use
- <symptom or task that should trigger this>

## Steps
1. <ordered action> — <completion criteria>
2. ...

## Gotchas
- <the thing a generic agent gets wrong here>

## Reference
- https://tsdevstack.dev/docs/<category>/<page>
```

Rules:

- Frontmatter is `name` + `description` only. No Claude-only fields.
- Link canonical detail to `tsdevstack.dev`; don't copy prose into the skill.
- Apply the no-op test: delete any line that doesn't change agent behavior
  versus its defaults.
