---
name: setup-spawnpoint
description: Connect this workspace to spawnpoint so deploys work. Use when the user wants to set up, connect, or configure spawnpoint, or when a spawnpoint tool (deploy_project) is missing.
---

# Connect to spawnpoint

Wire this workspace to spawnpoint so the `deploy_project` / `get_project` /
`list_projects` / `terminate_project` tools become available. Do this once. Drive it for the user: run the
commands yourself; don't just describe them. There are **no tokens to copy**: sign-in
happens in the browser via OAuth, and Claude Code stores the credentials itself.

> **Not installed yet?** If the user doesn't have the plugin at all, that's two commands
> first. They give you these skills plus the MCP wiring:
>
> ```bash
> curl -fsSL https://getspawnpoint.com/install | bash
> ```
>
> (Or by hand: `claude plugin marketplace add spawnpoint-inc/spawnpoint-plugin`, then
> `claude plugin install spawnpoint@spawnpoint`.)
>
> (From a local clone, use the repo root as the marketplace source instead of
> `spawnpoint-inc/spawnpoint-plugin`.) Then restart Claude Code and continue below.
>
> **Desktop app, no terminal habit?** You are Claude Code running inside the app, so you
> can run the installer yourself: run the `curl … | install` line above in a shell and
> the app's runtime picks up the new server on its next reload. The one step that stays
> the user's is the browser **Authenticate** click in section 4: that approval is a
> deliberate human-in-the-loop and cannot be automated.

## 0. Already registered? Check before anything else

If spawnpoint tools are already available in this session (`whoami`, `list_projects`,
`deploy_project`), the server is registered: call `whoami` and skip the rest of this
list except as follows.

- **It answers:** the workspace is connected. Say so, then do step 5.
- **401 / unauthorized:** the server is registered but not signed in. Go straight to
  step 4, then step 5. Do not re-register it.

Only when no spawnpoint tools exist at all, start at step 1.

## 1. Get the server URL

spawnpoint is a hosted service and its origin is `https://app.getspawnpoint.com`:
use that unless the user says otherwise. Developing against a locally running
spawnpoint is the exception, at `http://localhost:8080`. `getspawnpoint.com` itself is
the site and the installer, not an MCP endpoint: do not register it as the server URL.

## 2. Check the server is reachable

```bash
curl -fsS <URL>/healthz
```

If this fails, spawnpoint isn't running. For a local setup, tell the user to start it
(`./scripts/dev.sh` in the spawnpoint repo, in a separate terminal). For a hosted one,
double-check the URL. If the check cannot run at all (a sandbox, proxy, or firewall
blocks the request), say so and carry on with step 3: registering does not depend on
it, and `/mcp` shows whether the server answers.

## 3. Register the MCP server (the actual wiring)

```bash
claude mcp add --transport http --scope user spawnpoint <URL>/mcp
```

No header, no token: authentication is handled by OAuth in the next step. Use
`--scope user` so spawnpoint is available in every project, or `--scope project` to limit
it to this one.

> **Upgrading from an old token-based setup?** Remove the stale entry first:
> `claude mcp remove spawnpoint`, then re-add as above. (The old setup embedded an
> `Authorization` header; that's no longer needed and the token can be revoked in the
> console.)

## 4. Authenticate in the browser

Steps 4 and 5 are the user's to do, so put both in the same reply, as numbered
instructions, even when an earlier step failed.

Tell the user to **restart Claude Code** (or reconnect), then:

1. Run `/mcp`, select **spawnpoint**, and choose **Authenticate**.
2. A browser opens the spawnpoint consent page. If they're signed out, they sign in first
   (enter an email, then click the magic link spawnpoint emails them).
3. Click **Approve**. Claude Code receives the tokens and stores them securely itself
   (macOS Keychain); they refresh automatically, so this is a one-time step.

## 5. Turn on auto-update (one toggle, then never again)

Third-party plugin marketplaces do not auto-update by default, so a one-time toggle is
what keeps this plugin current from here on. Tell the user to run `/plugin`, open
**Marketplaces**, select **spawnpoint**, and enable **auto-update**. This is a UI
setting you cannot flip for them, so say it explicitly and say why: skills improve
often, and without the toggle they silently stay on today's version.

## 6. Verify

Confirm by asking to **list their spawnpoint projects**: if `list_projects` returns
without an auth error, they're connected and can deploy with the `deploy-to-spawnpoint`
skill.

## If something's off

- **connection refused / healthz fails**: spawnpoint isn't running; start it.
- **401 / unauthorized**: the grant was revoked or expired; run `/mcp` → spawnpoint →
  **Authenticate** again.
- **No browser opens**: make sure the spawnpoint server and the browser are reachable
  from this machine, then retry `/mcp` → Authenticate.
