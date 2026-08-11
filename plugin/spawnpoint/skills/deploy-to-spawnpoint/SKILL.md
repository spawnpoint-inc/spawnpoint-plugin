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
- `runtime`: `static` (HTML/CSS/JS), `node`, `python`, or `docker`. Default `static`.
  For `docker`, the bundle must include a `Dockerfile` at its root; the machine builds
  and runs it, and the container must listen on port 8080 (`PORT=8080` is set). Reach
  for it when the app is a compiled language or needs system packages; prefer the other
  runtimes when one fits, since they skip the build and start faster.
- `entrypoint`: for `node`/`python`, the file to run (default `index.js` / `main.py`).
  Omit for `static`, and **omit it for a `node` project that has a `start` script**: the
  deploy runs `npm start` and a `build` script, so Next.js, Remix, Astro, Nuxt,
  SvelteKit, and Express deploy as themselves. Set it only to override a wrong script.
- `files`: an array of `{ path, content }` for **every** file in the app. Inline files
  are for apps you just wrote; for files already on disk, use the upload path below
  and pass `upload_id` instead.
- `env`: optional environment variables.
- `visibility`: **new projects are `restricted` by default**: only the owner and emails
  they share with (verified by a one-time code) can open the link. Pass
  `visibility: "public"` when the user asks for a link anyone can open, or flip it later
  with `set_visibility`. Tell the user which they got. To let specific people into a
  restricted project, call `share_project` with each email: they get an invite email
  with the link, no account needed.

### Files on disk? Upload, don't inline

Regenerating existing files as inline `content` is slow and expensive: an 80 KB site
is minutes of token generation, on every redeploy. When the app's files already exist
on disk (or the bundle is more than a few small text files, or contains any binary
asset like an image), use the upload transport instead:

1. Call `create_upload` (no arguments). It returns `upload_id`, `upload_url`, and
   `command`: the exact one-liner to run.
2. Run the command from the app's directory. It tars the directory and uploads it in
   seconds: `tar czf - --exclude .git --exclude node_modules . | curl -fsS -T - "<upload_url>"`.
   Add more `--exclude` flags for anything else that should not deploy (build caches,
   `.env` files with secrets you don't want on the machine).
3. Call `deploy_project` with `upload_id` instead of `files`. Everything else (name,
   runtime, entrypoint, polling) works exactly the same.

The URL is single-use and expires in 30 minutes; a rejected archive (bad tar, `.git`
or `node_modules` inside) does not burn it, so fix the tar and re-run the command. If
`deploy_project` says the upload is unknown or expired, call `create_upload` again and
re-upload. The 20 MiB bundle limit applies on both paths.

### Rules

- **Keep it small**: files are injected into the machine (a few files / a few MB total).
  Inline assets where reasonable; don't include `node_modules` or build output.
- **Dependencies install automatically.** Ship `requirements.txt` (python) or
  `package.json` (node) at the bundle root and spawnpoint installs them on the machine
  before starting the app, so Flask, Express, and friends just work. Include
  `package-lock.json` when you have one (it gets the faster, reproducible `npm ci`).
  Don't vendor libraries into the bundle to avoid dependencies; declare them instead.
- **Images ride the upload path**: any image in the bundle means use `create_upload`
  (inline JSON is text, and large inline files are slow to generate). Still keep them
  web-sized for the page's sake: jpg or webp, not png, for anything photographic, and
  aim for **under 200 KB per image**. Generating an image yourself? Render it at the
  size it will be shown, not larger.
- **Node/Python must listen on port 8080** (the machine exposes `:8080`). Static sites are
  served automatically.
- After a successful call, close the loop (next section) until the project is `running`,
  then give the user the **`url`**: it is the project's permanent share link, it survives
  redeploys and machine replacement, and deep links (paths, query strings) work on it.
  Do not surface `machine_url` (the raw machine endpoint underneath): it is plumbing, it
  changes if the machine is replaced, and for restricted projects it refuses traffic.
  Mention they can view or terminate the project in their console:
  <https://app.spawnpoint.lol/console> (or `<origin>/console` for a dev setup).
- If the call returns an auth error, tell the user to run `/mcp`, select **spawnpoint**,
  and choose **Authenticate** (re-runs the browser OAuth flow). If the spawnpoint server
  isn't registered at all, run `/spawnpoint:setup-spawnpoint` first.

### Close the loop

`deploy_project` returns before the app is live: the status starts as `spawning` and
provisioning finishes in the background. Do not stop at the tool call; see the deploy
through:

- **Call `watch_deploy({ project_id })`.** It follows the deploy and returns when it
  settles, streaming each step (machine booting, SSH ready, pushing files, app live) as
  it happens. Relay those steps to the user as they arrive, so a two minute deploy does
  not look like a hang. This is the right tool here: polling in a loop is noisier and
  tells the user nothing while it waits.
- It returns as soon as the deploy ends, or after `wait_seconds` (default 180) if the
  deploy is still going. Still going? Call it again, and say so to the user.
- Only fall back to polling `get_project({ project_id })` every 15 seconds if
  `watch_deploy` is unavailable.
- `running`: done. Hand the user the URL.
- `sleeping` (seen on later checks, not during a deploy): the app was idle, so its
  machine is stopped and billing frozen. It is not broken. Visiting the share URL wakes
  it in about half a minute, and deploying to the same name wakes it too. Tell the user
  it is asleep, not down.
- `error`: the `error` field is the one-line reason; the actual cause is usually in the
  logs. Call `get_project_logs({ project_id })` and read the push log (upload, dependency
  install, and start output: a pip or npm failure prints its real error there). Fix what
  it shows and redeploy under the same name (a redeploy updates in place and keeps the
  URL). Explain to the user what happened in plain words.
- Still `spawning` after a `watch_deploy` call returns: call it again. Provisioning
  normally takes 1 to 3 minutes but can legitimately take up to about 12 on a slow
  machine allocation, so do not give up early; tell the user it is still provisioning
  and carry on.
- Running but misbehaving (blank page, 500s)? `get_project_logs({ project_id,
  kind: "runtime" })` fetches the app's live journal tail from the machine: read it,
  fix, redeploy.

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

Then call `watch_deploy({ "project_id": "proj_…" })`, relay its steps to the user, and
hand over the URL when it reports `running`.

## Related tools

- `watch_deploy({ project_id, wait_seconds? })`: follow a deploy and return when it
  settles, streaming each step. What to call after `deploy_project`.
- `get_project({ project_id })`: one project's status, URL, health, error reason, and the
  machine it runs on (node and python versions, vCPUs, memory). A single check rather
  than a wait.
- `get_project_logs({ project_id, kind? })`: the project's logs. `push` (default) is the
  last deploy's output; `runtime` is the app's live journal tail. The diagnosis tool.
- `set_visibility({ project_id, visibility })`: flip between `restricted` (owner +
  shared emails only; the default for new projects) and `public` (anyone with the
  link). Instant at the share URL; the machine reconfigures itself in seconds.
- `share_project({ project_id, email })`: let one person into a restricted project.
  They get an invite email with the link and verify with a one-time code sent to that
  address, no account needed. Call once per person; re-adding is harmless.
- `unshare_project({ project_id, email })`: revoke one viewer. Instant.
- `create_upload`: mint the single-use upload URL for the upload path above.
- `list_projects`: show the user's projects, URLs, and who each is shared with.
- `terminate_project({ project_id })`: take a project down.
- `schedule_teardown({ project_id, in? })`: schedule automatic termination (`in` like
  `45m`, `2h`, `1d`; omit `in` to cancel). `deploy_project` also accepts `teardown_in`
  directly. If the user calls the deploy temporary, a demo, or asks for it to go away
  later, set a teardown instead of relying on anyone remembering.

Docs: <https://spawnpoint.lol/mcp.html> covers all nine tools and their arguments.
