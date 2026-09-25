---
name: dollardeploy-cli
description: Use when deploying apps, managing hosts, or interacting with DollarDeploy infrastructure from the terminal. Triggers on "deploy", "ddc", "launch app", "create host", "provision server", "redeploy", "build app".
---

# DollarDeploy CLI (ddc)

Deploy apps to user-owned servers from the command line.

## Install

```bash
npm install -g @dollardeploy/cli
```

Or one-off: `npx @dollardeploy/cli <command>`

## Auth

```bash
# Interactive
ddc auth

# Non-interactive
ddc auth --api-key <key>

# For local dev server
ddc auth --api-key <key> --base-url http://localhost:3000
```

Key resolution: `--api-key` flag > `DOLLARDEPLOY_API_KEY` env > `~/.dollardeploy/auth` file.

## Quick Reference

| Task                  | Command                                               |
| --------------------- | ----------------------------------------------------- |
| List hosts            | `ddc host list [--json]`                              |
| Create host           | `ddc host create --name my-server --provider hetzner` |
| Prepare host          | `ddc host prepare <host-id>`                          |
| Destroy host          | `ddc host destroy <host-id> [--yes]`                  |
| Remove host (keep VM) | `ddc host remove <host-id> [--yes]`                   |
| Deploy from GitHub    | `ddc deploy --url <github-url> --hostId <host-id>`    |
| Deploy + new host     | `ddc deploy --url <github-url> --create-host`         |
| Deploy template       | `ddc deploy --template <id> --hostId <host-id>`       |
| Redeploy app          | `ddc deploy --appId <app-id>`                         |
| Build app             | `ddc build <app-id> [--deploy]`                       |
| List apps             | `ddc app list [--json]`                               |
| List templates        | `ddc template list [search]`                          |
| Add SSH key           | `ddc ssh add ~/.ssh/id_rsa --name my-key`             |
| Show user             | `ddc user`                                            |

## Deploy Workflow

1. **Push your code first:** The GitHub repo must be pushed to the branch before it can be built. DollarDeploy pulls from GitHub — local uncommitted changes won't be deployed.
2. **Ensure a host exists:** `ddc host list --json` — pick a host with `status: active`
3. **Deploy:** `ddc deploy --url https://github.com/org/repo --hostId <host-id>`
4. The CLI auto-detects existing apps by repo URL and redeploys instead of duplicating.

To deploy from a non-default branch, update the app's source branch:

```bash
ddc app modify <app-id> --sourceBranch <branch-name>
```

To set env vars or app properties during deploy:

```bash
ddc deploy --url <url> --hostId <id> --env:DATABASE_URL postgres://... --set:mainPort 8080
```

## Predefined & Customization Variables

DollarDeploy injects some env vars automatically and reads others you set to customize
build/deploy/service behavior. Set them via `--env NAME=VALUE` / `--env:NAME value` on deploy,
`ddc app modify <id> --env NAME=VALUE`, a template's `.dollardeploy.yml` `env:` map, or the dashboard.
**Booleans are `1`/`0`, never `true`/`false`.** Full reference: https://docs.dollardeploy.com/predefined-variables/

**Auto-provided (build + deploy — read, don't set):** `APP_URL` (`https://<hostname>`), `APP_HOSTNAME`,
`APP_ALIASES`, `APP_LISTEN_HOSTNAME`/`APP_INTERNAL_HOSTNAME` (`127.0.0.1` — bind here so the proxy fronts you),
`GIT_TAGS`, `GIT_LAST_COMMIT`, `NODE_ENV=production` (Node apps), `USER_EMAIL`, `USER_IPADDRESS`,
`PORT` (`mainPort`, else `env.PORT`, else `3000`). docker-compose also gets `USER_UID`/`USER_GID`.

**Service URLs (appear once the service is added):** `POSTGRES_URL` / `REDIS_URL` / `MONGODB_URL` / `MARIADB_URL`
(in-container variants `POSTGRES_DOCKER_URL`, `REDIS_DOCKER_URL`). Extract parts with
`${VAR:component}` — components: `host`, `hostname`, `port`, `path`, `username`, `password`, `database`, `query`, `fragment`
(e.g. `${POSTGRES_URL:password}`).

**Template generators (set in a template `env:`, generated only if referenced, then persisted & auto-secret):**
`${GENERATED_PWD}` (10-char), `${GENERATED_HASH}` (32-char), `${GENERATED_SECRET}` (64-char hex, `openssl rand -hex 32`).

**Customization toggles you set:**

| Variable                                                                       | Default       | Effect                                                                          |
| ------------------------------------------------------------------------------ | ------------- | ------------------------------------------------------------------------------- |
| `DEPLOY_HOSTNAME_MATCH`                                                        | `1`           | `0` skips the DNS-matches-server check (behind Cloudflare/CDN)                  |
| `APP_READY_TIMEOUT`                                                            | `300`         | Seconds to wait for the app to become ready                                     |
| `APP_HEALTHCHECK_ENABLE` / `APP_HEALTHCHECK_PATH` / `APP_HEALTHCHECK_EXTERNAL` | — / `/` / `0` | Enable healthchecks, path to probe, `1` probes `${APP_URL}`                     |
| `BUILD_NO_SERVICE_ENV`                                                         | —             | `1` excludes service URLs during build (e.g. Next.js prerender reaching the DB) |
| `BUILD_MEMORY_LIMIT` / `BUILD_CPU_LIMIT`                                       | —             | Build container resource caps                                                   |
| `DOCKER_COMPOSE_WAIT_TIMEOUT` / `DOCKER_COMPOSE_BUILD` / `DOCKER_COMPOSE_PULL` | `120` / — / — | Wait seconds; `1` rebuild each deploy; `policy` pull updated images             |

**Server/service install config (host env vars):** `POSTGRES_VERSION`, `POSTGRES_FORCE_INSTALL`,
`POSTGRES_DATABASES`, `POSTGRES_PASSWORD`, `POSTGRES_DATA_PATH`, `REDIS_DATA_PATH`, `ENCRYPTED_DEVICE` (LUKS).

Reserved build vars (`APP_NAME`, `APP_TYPE`, `GIT_URL`, and the auto-provided ones above) are rejected if you set them.

## Local Development

When testing against the local DollarDeploy dev server:

```bash
ddc auth --api-key <key> --base-url http://localhost:3000
# Or per-command:
ddc host list --base-url http://localhost:3000
# Or via env:
export DOLLARDEPLOY_BASE_URL=http://localhost:3000
```

## Provider Defaults

| Provider | `--type`    | `--region` | `--image`    |
| -------- | ----------- | ---------- | ------------ |
| hetzner  | cax11       | fsn1       | ubuntu-24.04 |
| do       | s-2vcpu-4gb | fra1       | ubuntu-24-04 |
| verda    | CPU.4V.16G  | FIN-01     | ubuntu-24.04 |

## Global Flags

- `--json` — machine-readable JSON output (stdout), progress goes to stderr
- `--verbose` / `-v` — verbose logging
- `--api-key <key>` — override stored auth
- `--base-url <url>` — override API URL (default: `https://dollardeploy.com`)

## When to Use CLI vs MCP Tools

- **CLI (`ddc`)**: Use for scripted/automated workflows, CI/CD, bash-based operations, or when the user explicitly asks to use the CLI.
- **MCP tools (`mcp__claude_ai_DollarDeploy__*`)**: Use for interactive conversational tasks within Claude Code when MCP server is connected.

Both talk to the same API. The CLI adds `--json` for piping into other tools.
