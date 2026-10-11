---
inclusion: manual
---

# Fork & Git Workflow

This is a fork of upstream `we-promise/sure`. The goal is to contribute PRs
upstream while keeping deployment-/workstation-specific changes out of those PRs
and out of upstream syncs.

Load this file on demand with `/context add .kiro/steering/fork-git-workflow.md`.

## Remotes

- `origin` → the fork (`stephenwiebe/sure`)
- `upstream` → `we-promise/sure`

## Branch roles

| Branch | Role | Rule |
| --- | --- | --- |
| `main` | Tracks upstream. Base for all upstream PRs. | Keep clean — never commit local-only changes here. |
| `local` | `main` + local-only dev overrides. The branch you run locally. | Deployment/workstation changes live here only. |
| `feat/*`, `fix/*` | Upstream-bound feature branches. | Branch off `main`, PR to upstream. |

Keeping `main` clean is what makes `--ff-only` upstream syncs painless and keeps
PR diffs free of local-only noise.

## What counts as local-only

Dev-environment and deployment changes that must be committed but should never go
upstream — e.g. `Procfile.dev`, `config/environments/development.rb`,
`config/database.yml` tweaks, `mise.toml`, `.kiro/`, local Tailwind CLI deps in
`package.json`. These live on `local`.

Prefer `.env.local` (gitignored) and the app's `SELF_HOSTED` / `app_mode` gating
over committed code changes whenever possible — no branch needed for those.

## Sync from upstream

```sh
git checkout main
git fetch upstream
git merge --ff-only upstream/main    # stays clean, never conflicts
git push origin main

git checkout local
git merge main                       # or: git rebase main for a tidy stack
```

## Contribute a PR upstream

Always branch off clean `main`, never off `local`:

```sh
git checkout main
git checkout -b fix/some-bug
# ...work, commit...
git push -u origin fix/some-bug
# open PR: fork fix/some-bug -> upstream main
```

## Guardrails

- Never commit local-only changes to `main`.
- Never base an upstream PR branch on `local` (it carries local commits).
- Keep local-only changes surface-level (new files, env vars, config toggles) to
  minimize recurring merge conflicts with upstream.
- Minimize local edits to core logic — those conflict on every sync.
