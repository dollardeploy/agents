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

| Task | Command |
|------|---------|
| List hosts | `ddc host list [--json]` |
| Create host | `ddc host create --name my-server --provider hetzner` |
| Prepare host | `ddc host prepare <host-id>` |
| Destroy host | `ddc host destroy <host-id> [--yes]` |
| Remove host (keep VM) | `ddc host remove <host-id> [--yes]` |
| Deploy from GitHub | `ddc deploy --url <github-url> --hostId <host-id>` |
| Deploy + new host | `ddc deploy --url <github-url> --create-host` |
| Deploy template | `ddc deploy --template <id> --hostId <host-id>` |
| Redeploy app | `ddc deploy --appId <app-id>` |
| Build app | `ddc build <app-id> [--deploy]` |
| List apps | `ddc app list [--json]` |
| List templates | `ddc template list [search]` |
| Add SSH key | `ddc ssh add ~/.ssh/id_rsa --name my-key` |
| Show user | `ddc user` |

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

| Provider | `--type` | `--region` | `--image` |
|----------|----------|------------|-----------|
| hetzner | cax11 | fsn1 | ubuntu-24.04 |
| do | s-2vcpu-4gb | fra1 | ubuntu-24-04 |
| verda | CPU.4V.16G | FIN-01 | ubuntu-24.04 |

## Global Flags

- `--json` — machine-readable JSON output (stdout), progress goes to stderr
- `--verbose` / `-v` — verbose logging
- `--api-key <key>` — override stored auth
- `--base-url <url>` — override API URL (default: `https://dollardeploy.com`)

## When to Use CLI vs MCP Tools

- **CLI (`ddc`)**: Use for scripted/automated workflows, CI/CD, bash-based operations, or when the user explicitly asks to use the CLI.
- **MCP tools (`mcp__claude_ai_DollarDeploy__*`)**: Use for interactive conversational tasks within Claude Code when MCP server is connected.

Both talk to the same API. The CLI adds `--json` for piping into other tools.
