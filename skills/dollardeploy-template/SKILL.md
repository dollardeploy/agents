---
name: dollardeploy-template
description: Use when creating, adding, or editing a DollarDeploy app template so an app appears in the deploy catalog at /r. Triggers on "create template", "add template", ".dollardeploy.yml", "dollardeploy template", "add app to catalog", "templates.json", "one-click deploy".
---

# Creating a DollarDeploy Template

A template is a one-click deployable app in the catalog (https://dollardeploy.com/r). You add a directory with a `.dollardeploy.yml` to the public **`github.com/dollardeploy/templates`** repo via a pull request. A GitHub Action then runs `index.js`, which scans every template dir, inlines any referenced files, and regenerates `templates.json` — the file the platform reads.

A template points at a source repo (its own or a third party's) and declares how to build and run it, what resources it needs, and which env vars to set. Unlike a [[dollardeploy-custom-service]] (which installs software on a host), a template deploys an **app**.

## Repo Conventions

| Item             | Rule                                                                                      |
| ---------------- | ----------------------------------------------------------------------------------------- |
| Directory        | One dir per template at the repo root, e.g. `ghost-cms/`                                  |
| Config file      | **Required**: `.dollardeploy.yml` (or `.yaml` / `.json`) in that dir                      |
| `id`             | Must match the directory name, lowercase, e.g. `id: ghost-cms`                            |
| Extra files      | `docker-compose.yml`, logos, scripts live alongside the config and are referenced by path |
| `templates.json` | **Generated** — never edit by hand; the Action rebuilds it on merge to `main`             |

## Config Schema

Top-level fields (`.dollardeploy.yml`):

| Field            | Req | Notes                                                                         |
| ---------------- | --- | ----------------------------------------------------------------------------- |
| `id`             | ✅  | Matches dir name                                                              |
| `name`           | ✅  | Display name                                                                  |
| `intro`          | ✅  | One-line summary for the card                                                 |
| `description`    |     | Longer markdown block. Shown on the template page                             |
| `logo`           |     | Absolute URL. For repo-hosted images use the raw githubusercontent URL        |
| `tags`           |     | List for filtering, e.g. `cms, oss, popular, template`                        |
| `deployTime`     |     | Human-friendly estimate, e.g. `~3 minutes`                                    |
| `demoUrl`        |     | Live demo link                                                                |
| `requirements`   |     | `memory` (MB), `cpu`, `storage` (GB), optional `gpu: {model, count}`          |
| `services`       |     | Host services to ensure first, e.g. `- docker` (needed for `docker-compose`)  |
| `postLaunchNote` |     | Markdown shown after deploy. Can reference app env, e.g. `${MINIO_ROOT_USER}` |
| `experimental`   |     | `true` to flag as experimental                                                |
| `introVideoUrl`  |     | Optional walkthrough video                                                    |

The `app` object (required):

| Field                                                     | Notes                                                                       |
| --------------------------------------------------------- | --------------------------------------------------------------------------- |
| `type`                                                    | **Required**. App type (see table below)                                    |
| `repositoryUrl`                                           | **Required**. Source repo to deploy                                         |
| `sourceBranch`                                            | Branch (or pin a tag), e.g. `main`                                          |
| `sourcePath`                                              | Subdir within the repo (for monorepos / this templates repo)                |
| `dockerComposeFile`                                       | Compose filename when `type: docker-compose` (default `docker-compose.yml`) |
| `mainPort`                                                | Port the app listens on; DollarDeploy proxies it to the public URL          |
| `env`                                                     | Map of env vars (supports interpolation, see below)                         |
| `buildCmd` / `startCmd` / `installCmd`                    | Override build/run/install commands (`native` type)                         |
| `preStartCmd` / `postStartCmd`                            | Shell run before/after start                                                |
| `files`                                                   | Inline files written into the source dir before build (see below)           |
| `buildPath` / `webPath` / `allowAccessFrom` / `backupCmd` | Advanced overrides                                                          |

## App Types

| `type`           | Use for                                                       |
| ---------------- | ------------------------------------------------------------- |
| `next`           | Next.js apps                                                  |
| `react`          | React apps (built, served)                                    |
| `react-static`   | Static React/SPA build                                        |
| `deno`           | Deno apps                                                     |
| `java`           | Java apps                                                     |
| `php`            | PHP apps                                                      |
| `native`         | Anything else — you supply `installCmd`/`buildCmd`/`startCmd` |
| `docker-compose` | Multi-container apps via a compose file                       |

## Interpolation Variables

Use `${VAR}` inside `env` values. The platform resolves these at launch

| Variable              | Resolves to                                                                     |
| --------------------- | ------------------------------------------------------------------------------- |
| `${APP_URL}`          | Full public URL, e.g. `https://app.example.com`                                 |
| `${APP_HOSTNAME}`     | Hostname only (no scheme)                                                       |
| `${PORT}`             | The app port (from `mainPort`, else `env.PORT`, else `3000`)                    |
| `${USER_EMAIL}`       | Deploying user's email                                                          |
| `${USER_IPADDRESS}`   | User's IP — typically with `allowAccessFrom: "${USER_IPADDRESS}"`               |
| `${GENERATED_PWD}`    | Random 10-char password — generated and saved in app env **only if referenced** |
| `${GENERATED_HASH}`   | Random 32-char hash — generated and saved in app env only if referenced         |
| `${GENERATED_SECRET}` | Random 32-byte hex (like `openssl rand -hex 32`) — generated only if referenced |

`GENERATED_*` values persist on the app's env after first launch, so secrets stay stable across redeploys. They are auto-marked secret in the UI. Others are just available for every app during build or deploy.

For the full set of predefined and customization variables (auto-provided app env, service URLs, `${VAR:component}` expansion, healthcheck/build/docker-compose toggles), see the env vars reference in [[dollardeploy-cli]] or https://docs.dollardeploy.com/predefined-variables/.

## docker-compose Conventions

Bind published ports to 127.0.0.1 so DollarDeploy's reverse proxy handles TLS and the public URL, and your app never exposed without reverse proxy:

```yaml
ports:
  - 127.0.0.1:$PORT:2368 # host loopback:PORT -> container port
```

Use `${VAR:-default}` for defaults so compose works standalone too. Declare named `volumes` for persistence. Always list `services: [docker]` in the template config.

## Example: docker-compose template

`ghost-cms/.dollardeploy.yml`

```yaml
id: ghost-cms
name: Ghost CMS
intro: "Independent platform for publishing by web and email newsletter."
logo: https://raw.githubusercontent.com/dollardeploy/templates/refs/heads/main/ghost-cms/logo.png
tags: [cms, blog, oss, popular]
deployTime: ~3 minutes
description: |
  Ghost is a powerful app for professional publishers to create, share,
  and grow a business around their content.
requirements:
  memory: 2048
  cpu: 2
  storage: 10
services:
  - docker
app:
  type: docker-compose
  repositoryUrl: https://github.com/dollardeploy/templates
  sourcePath: ghost-cms
  sourceBranch: main
  dockerComposeFile: docker-compose.yml
  env:
    SERVER_URL: ${APP_URL}
    MYSQL_PASSWORD: ${GENERATED_PWD}
    PORT: 3000
```

## Example: native template with inline files

For apps needing a setup script, ship it inline via `files` and run it with `preStartCmd` (no extra repo file needed):

```yaml
app:
  type: native
  repositoryUrl: https://github.com/dollardeploy/example-python-uv
  sourceBranch: main
  buildCmd: uv sync
  preStartCmd: bash ./setup.sh
  startCmd: uv run uvicorn main:app --host 0.0.0.0 --port 8000
  env:
    PORT: 8000
  files:
    - path: setup.sh
      content: |
        #!/bin/bash
        curl -LsSf https://astral.sh/uv/install.sh | sh
```

Templates also support `files: [{ path: relative/file }]` (no `content`) to inline a file that lives in the template dir.

## Publish Workflow

1. Add `templates/<id>/.dollardeploy.yml` (+ `docker-compose.yml`, logo, scripts as needed).
2. Validate it parses and matches the schema: from the templates repo root run `node index.js > /tmp/out.json` — it prints each template and errors on missing referenced files.
3. Open a PR against `github.com/dollardeploy/templates`.
4. On merge to `main`, the **Generate Templates** Action runs `node index.js > templates.json` and commits it; **Deploy Changed Templates** redeploys the demo. The app template then appears at `/r`.

## Common Mistakes

- **`id` ≠ directory name** → the entry is mislabeled / not found.
- **Editing `templates.json` by hand** → overwritten by the Action. Edit the `.dollardeploy.yml` instead.
- **`docker-compose` without `services: [docker]`** → host has no Docker, deploy fails.
- **Publishing the app's port directly** (not `127.0.0.1:$PORT:...`) → bypasses the proxy / no TLS.
- **Referencing a `files` path that doesn't exist** → `index.js` throws `Referenced file not found`.
- **Setting a reserved build var** (e.g. `APP_NAME`, `APP_TYPE`, `GIT_URL`) in `env` → launch rejects it.
- **Hardcoding secrets** instead of `${GENERATED_PWD}` / `${GENERATED_SECRET}` → not unique per deploy.
- **Private source repo** → the host can't clone it. Keep the source public.
