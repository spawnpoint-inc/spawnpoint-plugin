# spawnpoint plugin (Claude Code)

Install this in any Claude Code workspace and your agent can deploy small apps to
spawnpoint. It's **two skills** that drive the whole flow:

- **`setup-spawnpoint`**: connects the workspace (wires the MCP server; you sign in once
  in the browser via OAuth, no tokens to copy).
- **`deploy-to-spawnpoint`**: builds + deploys an app and hands you the link.

## Install

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

Manage or terminate projects at `http://localhost:8080/console`.

## Requirements

- The spawnpoint backend running and reachable (default `http://localhost:8080`). Same
  machine as the agent for a local setup; a hosted origin otherwise.

> Points at a **local** spawnpoint by default. For a hosted spawnpoint, give the setup skill
> your hosted origin (e.g. `https://spawnpoint.lol`) instead of `http://localhost:8080`.
