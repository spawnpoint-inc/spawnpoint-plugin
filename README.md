# spawnpoint plugin for Claude Code

Deploy the apps your AI agent builds and get a shareable link. This is the official
[Claude Code](https://claude.com/claude-code) plugin for
[spawnpoint](https://getspawnpoint.com): your agent builds a small app (a dashboard, a
tracker, an internal tool, a prototype), spawnpoint puts it online, and you get an HTTPS
link that is **private by default** and shared like a Google Doc. The people you send it
to never create an account.

## Install

One command wires Claude Code, Codex, Cursor, Copilot, and
[many more agents](https://getspawnpoint.com/agents.html):

```sh
curl -fsSL https://getspawnpoint.com/install | bash
```

Or install just this plugin by hand:

```sh
claude plugin marketplace add spawnpoint-inc/spawnpoint-plugin
claude plugin install spawnpoint@spawnpoint
```

Restart Claude Code so the skills load, then talk to your agent:

- **"set up spawnpoint"**: connects Claude Code to spawnpoint and walks you through a
  one-time browser sign-in. No keys to copy.
- **"deploy this to spawnpoint"**: checks the app, ships it, and hands you the link.
- **"share it with sam@example.com"** or **"make it public"**: decides who can open it.

## What's in it

Three skills, in [`plugin/spawnpoint/skills/`](plugin/spawnpoint/skills/):

- **`deploy-to-spawnpoint`**: picks the runtime (static, Node, Python, or Docker), uploads
  the app, follows the deploy until it is live, and hands back the permanent link.
  Next.js, Remix, Astro, and SvelteKit deploy as themselves from their package.json
  scripts; Python apps (Flask, FastAPI) run from their entrypoint.
- **`security-audit`**: checks an app before it ships. It finds leaked API keys and files
  that should not upload, checks the port and bind address spawnpoint needs, and flags
  debug mode, open CORS, and admin or upload routes with no login. It fixes what is safe
  to fix and stops the deploy on what is not. Run it yourself with
  `/spawnpoint:security-audit`.
- **`setup-spawnpoint`**: registers the MCP server and drives the OAuth sign-in.

The skills are A/B tested against a mocked spawnpoint server before each release, so
changes are measured, not guessed.

## Not using Claude Code?

spawnpoint is a remote MCP server. Any MCP client can connect to
`https://app.getspawnpoint.com/mcp` (Streamable HTTP, OAuth sign-in in the browser). It is
listed in the [official MCP registry](https://registry.modelcontextprotocol.io) as
`com.getspawnpoint/spawnpoint`. Per-agent steps: [supported agents](https://getspawnpoint.com/agents.html).

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

## Links

- Docs: [getspawnpoint.com](https://getspawnpoint.com), and
  [the MCP server and its tools](https://getspawnpoint.com/mcp.html)
- For agents: [llms.txt](https://getspawnpoint.com/llms.txt) and
  [llms-full.txt](https://getspawnpoint.com/llms-full.txt)
- Security reports: founders@getspawnpoint.com

This repo is the published plugin only; it is synced from spawnpoint's main repository on
every release.
