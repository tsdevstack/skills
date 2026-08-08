# tsdevstack skills

Agent skills for working in a tsdevstack app. Vendor-neutral: plain
[Agent Skills](https://agentskills.io) `SKILL.md` files, installable into any
supported coding agent (Claude Code, Cursor, Codex, Copilot, Gemini CLI, …).

## Install

```bash
# all tsdevstack skills, into your agent of choice
npx skills add tsdevstack/skills -a <your-agent>

# just one
npx skills add tsdevstack/skills --skill tsdevstack-add-an-endpoint

# browse first
npx skills add tsdevstack/skills --list
```

Update later with `npx skills update`.

## What's here

Skills are grouped by trigger into three folders:

- `cross-cutting/` — `tsdevstack-auth-model` (two-layer Kong + AuthGuard) ·
  `tsdevstack-nest-common` (backend building blocks) · `tsdevstack-secrets`
  (the 3-file secret system) · `tsdevstack-messaging` (pub/sub) ·
  `tsdevstack-storage` (object storage).
- `local/` — `tsdevstack-run-locally` (bring-up + troubleshooting) ·
  `tsdevstack-add-an-endpoint` (decorator → OpenAPI → Kong → client) ·
  `tsdevstack-db-changes` (Prisma migrations: local creates, deploys apply) ·
  `tsdevstack-service-and-worker-lifecycle` · `tsdevstack-frontend-bff` (Next.js BFF).
- `cloud/` — `tsdevstack-command-model` (what each command subsumes) ·
  `tsdevstack-deploy-troubleshoot`.
- `curation.json` — recommended third-party skills, pinned and pulled from each
  author's repo (**not re-hosted here**); decisions recorded under
  `install` / `candidates` / `rejected`.

## This is NOT `.agents/skills/`

These are the **shippable, user-facing** skills that publish to `tsdevstack/skills`
and get installed into user projects. They are distinct from the repo-root
`.agents/skills/`, which are skills for developing tsdevstack itself. Don't
cross-wire them.

## Authoring conventions

- One skill per directory, each with a single `SKILL.md`.
- **Portable frontmatter only:** `name` and `description`. No Claude-only fields
  (`model`, `effort`, `context`, `allowed-tools`, `disable-model-invocation`) —
  other vendors ignore them, so never make behavior depend on them.
- Keep bodies thin. Put canonical detail on the docs site and **link to the AI
  `.md` version of each page**: `https://tsdevstack.dev/docs/<category>/<page>.md`
  (rspress emits a per-page `.md` for agents). One source of truth, fetchable by
  any agent.
- `description` should front-load the leading word and list one trigger per
  distinct situation.
- See `TEMPLATE.md` for the skeleton.

## How it ships

Authored here in the monorepo (`skills/`) and mirrored to the public repo
`tsdevstack/skills` by
[`.github/workflows/framework-sync-repos.yml`](../.github/workflows/framework-sync-repos.yml).
Edit here, then run the **Sync to Individual Repos** workflow to publish.
Consumers pick up changes via `npx skills update`.