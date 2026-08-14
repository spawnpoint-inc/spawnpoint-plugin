# spawnpoint plugin

The official [Claude Code](https://claude.com/claude-code) plugin for
[spawnpoint](https://getspawnpoint.com): your agent builds a small app, spawnpoint puts it
online, and you get a link to share.

## Install

```sh
claude plugin marketplace add spawnpoint-inc/spawnpoint-plugin
claude plugin install spawnpoint@spawnpoint
```

Restart Claude Code so the skills load, then just talk to your agent:

- **"set up spawnpoint"**: connects Claude Code to spawnpoint and walks you through a
  one-time browser sign-in. No keys to copy.
- **"deploy this to spawnpoint"**: builds, ships, and hands you the link.

## Update

**Do this once and never think about updates again:** run `/plugin`, open
**Marketplaces**, select **spawnpoint**, and enable **auto-update**. Third-party
marketplaces have it off by default, which is why it takes the one toggle; after that
Claude Code picks up new versions on its own.

Prefer manual? Two commands, then restart Claude Code:

```sh
claude plugin marketplace update spawnpoint
claude plugin update spawnpoint@spawnpoint
```

New MCP tools need no plugin update at all: agents discover them from the server.

## What's in it

Two skills and the MCP wiring, in [`plugin/spawnpoint/`](plugin/spawnpoint/):

- `setup-spawnpoint`: registers the MCP server and drives the OAuth sign-in.
- `deploy-to-spawnpoint`: packages the app and calls the `deploy_project` tool.

Details in the [plugin README](plugin/spawnpoint/README.md). This repo is the published
plugin only; docs live at [getspawnpoint.com](https://getspawnpoint.com).
