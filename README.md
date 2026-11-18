# Miabi stacks

GitOps manifests (`miabi.io/v1`) for this workspace, one directory per stack. Each directory is
its own Miabi Git source, on branch `main` with the directory as the path, so each stack syncs,
prunes and reports on its own.

| Directory | Stack | Git source path |
|---|---|---|
| [`n8n/`](n8n/README.md) | n8n in queue mode: editor, worker, task runners, managed Postgres + Redis | `n8n` |
| [`paperclip/`](paperclip/README.md) | Paperclip: AI agent companies, on managed Postgres | `paperclip` |

## Rules for this repository

- **No YAML at the root.** A source parses every `.yaml`/`.yml` under its path. Anything that
  isn't a `miabi.io/v1` manifest fails the sync, as `docker-compose.yml` once did.
- **One source per directory.** A resource belongs to the source that created it; another
  source trying to change it is refused. Don't point two sources at the same directory, and
  don't point one at the root.
- **New stack:** add a directory, then create a Git source with that path.
- **Names are workspace-wide.** Prefix every resource with its stack's name (`n8n-*`,
  `paperclip-*`) so stacks never collide.
- Each stack's README covers its first-sync steps.
