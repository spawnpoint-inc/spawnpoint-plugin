# spawnpoint plugin (Claude Code)

Install this in any Claude Code workspace and your agent can deploy small apps to
spawnpoint. It's **two skills** that drive the whole flow:

- **`setup-spawnpoint`**: connects the workspace (wires the MCP server; you sign in once
  in the browser via OAuth, no tokens to copy).
- **`deploy-to-spawnpoint`**: builds + deploys an app and hands you the link.

## Install

```bash
curl -fsSL https://spawnpoint.lol/install | bash
```

Or by hand:

```bash
claude plugin marketplace add spawnpoint-inc/spawnpoint-plugin
claude plugin install spawnpoint@spawnpoint
```

From a local clone, point at the **repo root** (that's where
`.claude-plugin/marketplace.json` lives, not this `plugin/` directory):

```bash
claude plugin marketplace add /path/to/spawnpoint
claude plugin install spawnpoint@spawnpoint
```

Restart Claude Code afterwards so the skills load.

## Use it: all skill-driven

Just talk to the agent:

1. **"set up spawnpoint"**
   It wires the MCP connection for you. Restart Claude Code, run `/mcp` → spawnpoint →
   **Authenticate**, and approve in the browser. Claude Code keeps the credentials in your
   OS keychain and refreshes them automatically: no tokens to copy, ever.
2. **"deploy this to spawnpoint"** (or "build me a small X and deploy it")
   It ships the app and gives you a public link.

Manage or terminate projects in your spawnpoint console: `<your spawnpoint origin>/console`,
which is `http://localhost:8080/console` for a local setup.

## Requirements

- The spawnpoint backend running and reachable (default `http://localhost:8080`). Same
  machine as the agent for a local setup; a hosted origin otherwise.

> Points at a **local** spawnpoint by default: spawnpoint runs on your own machine today.
> For a hosted spawnpoint, give the setup skill that origin instead.

## Docs

Full documentation lives at [spawnpoint.lol](https://spawnpoint.lol):

- [Getting started](https://spawnpoint.lol/getting-started.html): install, sign in, deploy.
- [MCP server](https://spawnpoint.lol/mcp.html): the endpoint and its three tools.
- [API tokens](https://spawnpoint.lol/api-tokens.html): `spk_live_` credentials for clients
  that cannot do OAuth.
- [Security](https://spawnpoint.lol/security.html) and
  [Terms](https://spawnpoint.lol/terms.html).

Questions: [founders@spawnpoint.lol](mailto:founders@spawnpoint.lol).
