---
name: dollardeploy-custom-service
description: Use when creating, packaging, or publishing a custom DollarDeploy service repo to install software on a host (e.g. tailscale, a queue, an exporter). Triggers on "custom service", "service-<name> repo", "prepare.sh service", "install X on the host via DollarDeploy", "{{env:SERVICE_CUSTOM}}".
---

# Creating a DollarDeploy Custom Service

A custom service installs extra software on a DollarDeploy-managed host. You publish a **public** GitHub (or Gist) repo; the user pastes its URL on the host's Services tab. On every host _prepare_ (when applying pending changes), DollarDeploy clones the repo and runs its `prepare.sh` on the server.

Built-in services (redis, postgres, mariadb, mongodb, docker, firewall, luks, ddagent) are part of the platform. A custom service is anything else, shipped as your own repo — no code changes to DollarDeploy needed.

## Repo Conventions

| Item           | Rule                                                                                                        |
| -------------- | ----------------------------------------------------------------------------------------------------------- |
| Repo name      | `service-<name>`, e.g. `service-tailscale`                                                                  |
| Visibility     | **Public** — host clones with no credentials; private repos fail to clone                                   |
| Host           | `github.com/...` or `gist.github.com/...` (only these are accepted)                                         |
| `README.md`    | First `# heading` is the service name — **lowercase**, e.g. `# tailscale`.                                  |
|                | Becomes the on-host folder `~/services/<name>`. Falls back to URL/id if missing                             |
| `prepare.sh`   | **Required**, repo root. Runs on every prepare. Must be idempotent                                          |
| `uninstall.sh` | Optional, repo root. Runs once when the service is removed, then the dir is deleted                         |
| -------------- | ----------------------------------------------------------------------------------------------------------- |

## prepare.sh Contract

- Runs **as the app user** (not root) from inside the cloned dir, on Ubuntu/Debian. Use `sudo` for root actions.
- `SERVICE_ID` is exported, plus the host's full deploy env (host env vars, `HOST_ID`, etc.).
- Re-runs on every prepare — guard installs (`command -v`, file checks). Non-zero exit **fails the whole host prepare**, so `set -e` and fail loudly on real errors.
- **Return values** to DollarDeploy by echoing markers on their own line. They are stored on the service and re-injected as env on the next prepare (so generated secrets persist):

  ```bash
  # Saved in the service env vars, populates app env as TAILSCALE_URL
  echo "{{env:SERVICE_CUSTOM_${SERVICE_ID}_TAILSCALE_URL:http://127.0.0.1:8080}}"
  ```

  The `SERVICE_CUSTOM_${SERVICE_ID}_` prefix is mandatory — only prefixed markers will be persisted for the service. 
  On the next prepare they come back as `SERVICE_CUSTOM_${SERVICE_ID}_API_URL` in the environment; read them to stay idempotent.

## Example: service-tailscale

`README.md`

```markdown
# tailscale

Installs the Tailscale agent and brings the host onto your tailnet.
```

`prepare.sh`

```bash
#!/bin/bash
set -e

# Idempotent: skip if already installed
if ! command -v tailscale >/dev/null 2>&1; then
  echo "Installing tailscale"
  curl -fsSL https://tailscale.com/install.sh | sudo sh
fi

# Tailscale auth key (or OAuth client secret). Set as a host env var.
if [ -z "${TAILSCALE_AUTH_SECRET:-}" ]; then
  echo "tailscale: TAILSCALE_AUTH_SECRET is required (set it as a host env var)"
  exit 1
fi

sudo tailscale up --authkey "$TAILSCALE_AUTH_SECRET"

# Report the tailnet IP back to DollarDeploy
TS_IP=$(tailscale ip -4 2>/dev/null | head -n1 || true)
if [ -n "$TS_IP" ]; then
  echo "{{env:SERVICE_CUSTOM_${SERVICE_ID}_TAILSCALE_IP:$TS_IP}}"
fi
```

`uninstall.sh`

```bash
#!/bin/bash
set -e
sudo tailscale down || true
sudo apt-get purge -y tailscale || true
```

## Add the service to the host

1. Push the repo public to `github.com/<owner>/service-<name>`.
2. Host → **Services** tab → add a service of type **Custom** → paste the repo URL. Or via CLI/API with `type: custom`, `customUrl: <repo-url>`.
3. **Prepare** the host. The repo clones into `~/services/<name>`, `prepare.sh` runs, the service flips to `active`, and emitted env is stored.
4. To remove: delete the service, then prepare again — `uninstall.sh` runs and the dir is removed.

A host can hold many custom services, but duplicate URLs on one host are rejected.

## Common Mistakes

- **Private repo** → host clone fails. Keep it public.
- **No `prepare.sh`** at the repo root → prepare errors out.
- **Forgetting the `SERVICE_CUSTOM_${SERVICE_ID}_` prefix** on markers → values are dropped, not saved.
- **Non-idempotent script** → re-installs or errors on the second prepare. Always guard.
- **README heading not lowercase / not first** → wrong folder name or fallback to the raw id.
