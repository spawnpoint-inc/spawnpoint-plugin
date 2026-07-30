---
name: deploy-to-spawnpoint
description: Deploy a small app to spawnpoint and get a public link. Use when the user asks to deploy, ship, or launch an app via spawnpoint.
---

# Deploy to spawnpoint

Deploy a small app to spawnpoint. It runs the app on managed infrastructure and returns a
public URL. Use the `deploy_project` MCP tool.

## When to use

The user wants to deploy / ship / launch / publish a small app, dashboard, tool, or
prototype, and wants a shareable link.

## How to deploy

Call `deploy_project` with:

- `name`: a short slug for the project (e.g. `standup-board`).
- `runtime`: `static` (HTML/CSS/JS), `node`, or `python`. Default `static`.
- `entrypoint`: for `node`/`python`, the file to run (default `index.js` / `main.py`).
  Omit for `static`.
- `files`: an array of `{ path, content }` for **every** file in the app.
- `env`: optional environment variables.

### Rules

- **Keep it small**: files are injected into the machine (a few files / a few MB total).
  Inline assets where reasonable; don't include `node_modules` or build output.
- **Node/Python must listen on port 8080** (the machine exposes `:8080`). Static sites are
  served automatically.
- After a successful call, give the user the returned **`url`**, and mention they can view
  or terminate the project in their console: `<spawnpoint origin>/console`, which is
  <http://localhost:8080/console> for a local setup.
- If the call returns an auth error, tell the user to run `/mcp`, select **spawnpoint**,
  and choose **Authenticate** (re-runs the browser OAuth flow). If the spawnpoint server
  isn't registered at all, run `/spawnpoint:setup-spawnpoint` first.

### Example

```
deploy_project({
  "name": "hello",
  "runtime": "static",
  "files": [
    { "path": "index.html", "content": "<!doctype html><h1>hello from spawnpoint</h1>" }
  ]
})
→ { "project_id": "proj_…", "url": "https://…", "status": "running" }
```

## Related tools

- `list_projects`: show the user's projects and URLs.
- `terminate_project({ project_id })`: take a project down.

Docs: <https://spawnpoint.lol/mcp.html> covers all three tools and their arguments.
