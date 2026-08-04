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
- **Keep images small**: every file travels inline over the deploy transport, and large
  images are the thing that breaks it. Resize and compress any image before bundling
  (jpg or webp, not png, for anything photographic): aim for **under 200 KB per image**,
  and never ship one over 500 KB. Generating an image yourself? Render it at the size it
  will be shown, not larger.
- **Node/Python must listen on port 8080** (the machine exposes `:8080`). Static sites are
  served automatically.
- After a successful call, close the loop (next section) until the project is `running`,
  then give the user the **`url`**, and mention they can view or terminate the project in
  their console: `<spawnpoint origin>/console`, which is
  <http://localhost:8080/console> for a local setup.
- If the call returns an auth error, tell the user to run `/mcp`, select **spawnpoint**,
  and choose **Authenticate** (re-runs the browser OAuth flow). If the spawnpoint server
  isn't registered at all, run `/spawnpoint:setup-spawnpoint` first.

### Close the loop

`deploy_project` returns before the app is live: the status starts as `spawning` and
provisioning finishes in the background. Do not stop at the tool call; see the deploy
through:

- Poll `get_project({ project_id })` with the returned id, roughly every 15 seconds.
- `running`: done. Hand the user the URL.
- `error`: read the `error` field, explain the failure to the user in plain words, fix
  what is fixable in the bundle, and redeploy under the same name (a redeploy updates
  in place and keeps the URL).
- Still `spawning`: keep polling. Provisioning normally takes 1 to 3 minutes but can
  legitimately take up to about 12 on a slow machine allocation, so do not give up
  early; if it runs long, tell the user it is still provisioning and carry on.

### Social preview (optional)

A spawnpoint link pasted into a chat unfurls nicer with Open Graph tags, and the
console generates its own live thumbnail either way, so none of this is required.
When the app serves HTML and has no tags, adding `og:title` (the app's name),
`og:description` (one plain sentence), and `<meta name="twitter:card" content="summary">`
to `<head>` is cheap polish: do it in the first deploy, no follow-up needed.

- `og:image` is extra credit, not a requirement: only add it when the user cares about
  a picture in link unfurls AND the app has a suitable raster image (jpg, near
  1200x630, **under 200 KB**). It needs an absolute URL, which exists only after the
  first deploy: deploy, fill it in, redeploy under the same name. Do not do this
  dance by default.
- Never replace tags the app already has: existing tags mean someone chose them.

### Example

```
deploy_project({
  "name": "hello",
  "runtime": "static",
  "files": [
    { "path": "index.html", "content": "<!doctype html><h1>hello from spawnpoint</h1>" }
  ]
})
→ { "project_id": "proj_…", "url": "https://…", "status": "spawning" }
```

Then poll `get_project({ "project_id": "proj_…" })` until the status is `running`.

## Related tools

- `get_project({ project_id })`: one project's status, URL, health, and error reason.
  The poll target after a deploy.
- `list_projects`: show the user's projects and URLs.
- `terminate_project({ project_id })`: take a project down.

Docs: <https://spawnpoint.lol/mcp.html> covers all four tools and their arguments.
