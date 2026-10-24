# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

The web UI for Tranga: a Nuxt 4 / Vue app with Nuxt UI, in `website/`, served as a static site by nginx (`Dockerfile`, `nginx/`). nginx proxies `/api` to `$API_URL`. The API is the fork Nirvana77/tranga (`/mnt/Data/git/tranga`).

## Branches

This checkout is the fork Nirvana77/tranga-website. `main` mirrors upstream C9Glax/tranga-website and must stay free of our own commits. Our work goes on `develop`, the default branch. `.github/workflows/sync-develop.yml` merges `main` into `develop` after upstream updates. The CronJob `k8s/Tranga/prod/50-fork-sync.yaml` fast-forwards `main` from upstream daily.

## API client

`nuxt-open-fetch` generates the typed `$api`/`useApi` client from the OpenAPI document of the API fork's `develop` branch (`openFetch` in `website/nuxt.config.ts`, fetched over raw GitHub at build time). A new API field is only available here after it is pushed to Nirvana77/tranga `develop`. Types come from `components['schemas'][...]` in `#open-fetch-schemas/api`.

## Checks (Docker only, no Node on the host)

```sh
mkdir -p ~/.cache/tranga-npm-cache
nd() { docker run --rm -u "$(id -u):$(id -g)" -e HOME=/tmp/h -e npm_config_cache=/cache -e npm_config_update_notifier=false \
  -v ~/.cache/tranga-npm-cache:/cache -v "$PWD/website":/app -w /app node:24-alpine sh -c "mkdir -p /tmp/h; $*"; }
nd 'npm ci --no-audit --no-fund && npx nuxt prepare'   # once; npm blocks the postinstall that runs nuxt prepare
nd 'npx eslint app && npx prettier --check app'
nd 'npx -y -p vue-tsc@2.2.10 -p typescript@5.8.3 vue-tsc --noEmit -p .nuxt/tsconfig.app.json'
```

Type-check against `.nuxt/tsconfig.app.json`: the root `tsconfig.json` only has project references, so `-p tsconfig.json` checks nothing. `nuxi typecheck` fails without saying why.

Style: Prettier (`website/.prettierrc`), 4-space indent, single quotes.

## Deploying

`k8s/Tranga/build.sh --website` builds `develop` on the cluster's buildkitd and switches the `tranga-website` Deployment (`--website <ref>` or `--website --local` also work). The build logs `404 /api/v2/...` errors while prerendering; that's expected, since no API exists at build time (`nitro.prerender.failOnError: false`).
