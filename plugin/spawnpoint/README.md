# spawnpoint plugin (Claude Code)

Install this in any Claude Code workspace and your agent can deploy small apps to
spawnpoint. It's **two skills** that drive the whole flow:

- **`setup-spawnpoint`**: connects the workspace (wires the MCP server; you sign in once
  in the browser via OAuth, no tokens to copy).
- **`deploy-to-spawnpoint`**: builds + deploys an app and hands you the link.

## Install

```bash
curl -fsSL https://getspawnpoint.com/install | bash
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
which is <https://app.getspawnpoint.com/console> for the hosted service.

## Requirements

- A reachable spawnpoint origin. spawnpoint is a hosted service; use its origin when you
  have one. For development, a locally running spawnpoint works the same way (default
  `https://app.getspawnpoint.com`; a dev instance runs at `http://localhost:8080`).

> The setup skill asks for the origin, so hosted and local setups differ only in the URL
> you give it.

## Docs

Full documentation lives at [getspawnpoint.com](https://getspawnpoint.com):

- [Getting started](https://getspawnpoint.com/getting-started.html): install, sign in, deploy.
- [MCP server](https://getspawnpoint.com/mcp.html): the endpoint and its five tools.
- [API tokens](https://getspawnpoint.com/api-tokens.html): `spk_live_` credentials for clients
  that cannot do OAuth.
- [Security](https://getspawnpoint.com/security.html) and
  [Terms](https://getspawnpoint.com/terms.html).

Questions: [founders@getspawnpoint.com](mailto:founders@getspawnpoint.com).
