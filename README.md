# spawnpoint plugin

The official [Claude Code](https://claude.com/claude-code) plugin for
[spawnpoint](https://spawnpoint.lol): your agent builds a small app, spawnpoint puts it
online, and you get a link to share.

## Install

```sh
claude plugin marketplace add spawnpoint-inc/spawnpoint
claude plugin install spawnpoint@spawnpoint
```

Restart Claude Code so the skills load, then just talk to your agent:

- **"set up spawnpoint"**: connects Claude Code to spawnpoint and walks you through a
  one-time browser sign-in. No keys to copy.
- **"deploy this to spawnpoint"**: builds, ships, and hands you the link.

## What's in it

Two skills and the MCP wiring, in [`plugin/spawnpoint/`](plugin/spawnpoint/):

- `setup-spawnpoint`: registers the MCP server and drives the OAuth sign-in.
- `deploy-to-spawnpoint`: packages the app and calls the `deploy_project` tool.

Details in the [plugin README](plugin/spawnpoint/README.md). This repo is the published
plugin only; docs live at [spawnpoint.lol](https://spawnpoint.lol).
