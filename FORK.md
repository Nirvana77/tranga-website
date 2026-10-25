# Nirvana77/tranga-website: what this fork adds

This is a fork of [C9Glax/tranga-website](https://github.com/C9Glax/tranga-website), the web UI for Tranga. It's built against the API fork [Nirvana77/tranga](https://github.com/Nirvana77/tranga). That repo's [FORK.md](https://github.com/Nirvana77/tranga/blob/develop/FORK.md) describes the API side.

## Branches

`main` mirrors upstream; our changes are on `develop` (the default branch).
- The k8s CronJob `Tranga/prod/50-fork-sync.yaml` fast-forwards `main` daily.
- `.github/workflows/sync-develop.yml` then merges `main` into `develop`.

Deploy with `k8s/Tranga/build.sh --website`.

## Changes

- **API schema from the fork:** the typed API client is generated from Nirvana77/tranga `develop`, not upstream's `testing` branch (`website/nuxt.config.ts`). That's how the fork's new API fields reach the UI.
- **Status bar:** shown at the bottom of every page.
  - It shows what Tranga is doing right now (the current download with progress, plus running and queued counts).
  - Click it for every running, queued and scheduled task.
  - It polls `/v2/Worker` every 3 seconds.
- **Priorities and limits:**
  - The manga page has a priority (High/Normal/Low).
  - Settings → Downloads & tasks has the number of tasks and chapter downloads at the same time, and the order kinds of tasks start in.
  - The status bar shows running / limit.
- **Chapter language:** a badge next to each chapter's website icon (EN, IT…).
- **Download language setting:** Settings → Downloads & tasks.
  - Pick the language chapters are downloaded in, from the languages your enabled websites offer.
  - It also shows which websites have the selected language.
- **Docker build fix:** upstream's Dockerfile upgraded npm, and the new npm rejects the `-dd` flag used in the next step. The fork installs with `npm ci` from the lockfile.

## Development

See [CLAUDE.md](CLAUDE.md) for the Docker-only lint and type-check commands.
