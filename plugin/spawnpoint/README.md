# spawnpoint plugin

spawnpoint is the easiest way to bring your software online. With this plugin, Claude
can put an app it built for you online and hand you a link, in a chat, in Cowork, or in
Claude Code. The link is private until you share it with someone's email address.

The plugin brings the spawnpoint connector (the tools Claude uses to deploy) and three
skills that tell Claude how to use them:

- **`deploy-to-spawnpoint`**: deploys an app and hands you the link, and walks you
  through signing in the first time. You sign in once in the browser; there are no keys
  to copy.
- **`security-audit`**: checks an app before it ships (secrets, files that should not
  upload, the port and bind address spawnpoint needs, debug mode, open admin, upload,
  or presign routes, private files in a public bucket), fixes what is safe to fix, and
  stops the deploy on what is not.
- **`open-models`**: when the app calls OpenAI or Anthropic, asks once whether to run it
  on open models through spawnpoint instead (no provider account, no key), picks the
  model, and makes the one-line edit.

## Install in claude.ai or the Claude desktop app

1. Open **Customize**, then **Plugins**, then **Add**, then **Add marketplace**, and enter
   `https://github.com/spawnpoint-inc/spawnpoint-plugin`.
2. Install **spawnpoint**, open its **Connectors** tab, and click **Connect**.
3. Sign in to spawnpoint in the browser that opens and click **Approve**.

The plugin also reaches Claude Code on the same account the next time it starts.

## Install in Claude Code

```bash
curl -fsSL https://getspawnpoint.com/install | bash
```

The installer also sets up your other coding agents. Or by hand:

```bash
claude plugin marketplace add spawnpoint-inc/spawnpoint-plugin
claude plugin install spawnpoint@spawnpoint
```

Restart Claude Code, run `/mcp`, select **spawnpoint**, choose **Authenticate**, and
approve in the browser. Claude Code keeps the sign-in and refreshes it itself.

From a local clone of the app repo, point at the **repo root** (that's where
`.claude-plugin/marketplace.json` lives, not this `plugin/` directory):
`claude plugin marketplace add /path/to/spawnpoint`.

## Use it

Ask Claude to build something and put it online: "make a page where my running club
says who is coming on Saturday, and deploy it with spawnpoint." Claude builds the app,
deploys it, and gives you the link. Ask it to share the link with a teammate's email,
to change the app, or to take it down.

Your apps are listed in the console at <https://app.getspawnpoint.com/console>.

## Data

When you deploy, Claude sends spawnpoint the app's files, its name, and the settings
and secrets you give it, over HTTPS to app.getspawnpoint.com, signed in as you.
spawnpoint stores them to run the app. When you share an app, spawnpoint emails the
address you give it. The `security-audit` skill can run `npm audit`, `pip-audit`, or
`osv-scanner` on your computer, which sends the app's list of dependencies to the npm
registry or the OSV vulnerability database. The
[privacy policy](https://getspawnpoint.com/privacy.html) has the details.

## Docs

- [Getting started](https://getspawnpoint.com/getting-started.html): install, sign in, deploy.
- [MCP server](https://getspawnpoint.com/mcp.html): the endpoint and every tool.
- [Security](https://getspawnpoint.com/security.html) and
  [Terms](https://getspawnpoint.com/terms.html).

Questions: [founders@getspawnpoint.com](mailto:founders@getspawnpoint.com).
