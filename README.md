# DollarDeploy Agents

Open repository to make it easy for AI agents to manage your infrastructure and deploy your apps using [DollarDeploy](https://dollardeploy.com).

## Skills

| Skill | Description |
|-------|-------------|
| `dollardeploy-cli` | Deploy apps, manage hosts, and interact with DollarDeploy from the terminal using the [`ddc` CLI](https://www.npmjs.com/package/@dollardeploy/cli) |

## Claude Code

You can register this repository as a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) Plugin marketplace by running the following command in Claude Code:

`/plugin marketplace add dollardeploy/agents`

**Then, to install a specific set of skills:**

- Select Browse and install plugins
- Select **dollardeploy-agents**
- Select **dollardeploy-skills**
- Select **Install now**

**Alternatively, directly install either Plugin via:**

`/plugin install dollardeploy-skills@dollardeploy-agents`

After installing the plugin, you can use the skill by just mentioning it or asking to deploy your app.

## Prerequisites

Install the DollarDeploy CLI:

```bash
npm install -g @dollardeploy/cli
```

Then authenticate:

```bash
ddc auth
```

Get your API key from [DollarDeploy Settings](https://dollardeploy.com/settings/api).

## Links

- [DollarDeploy](https://dollardeploy.com) - Platform
- [CLI on npm](https://www.npmjs.com/package/@dollardeploy/cli) - `@dollardeploy/cli`
- [Documentation](https://docs.dollardeploy.com/cli) - CLI docs
- [API Reference](https://dollardeploy.com/apidocs) - REST API
- [Discord](https://discord.gg/BHbfmaAGb7) - Community
- [Claude Code Plugins](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills) - How plugins work
