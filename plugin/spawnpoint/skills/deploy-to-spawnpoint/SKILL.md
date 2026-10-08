---
name: deploy-to-spawnpoint
description: Deploy a small app to spawnpoint and get a public link. Use when the user asks to deploy, ship, or launch an app via spawnpoint.
---

# Deploy to spawnpoint

spawnpoint runs a small app on its own machine and returns a permanent share link.

## The workflow

1. **Check it is safe to ship.** An app that was already on disk (code you did not
   write in this session): run the `security-audit` skill on its directory first,
   use its exclusions as `tar --exclude` flags, and do not deploy while it reports a
   block. An app you write yourself: hold to these as you write it: secrets only in
   `env`, never echoed back; read `PORT` and bind `0.0.0.0`; no debug mode; a token
   check on admin and owner-only upload or presign routes.
2. **Pick the transport.** Files already on disk, more than a few small text files, or
   any image: call `create_upload`, run the `command` it returns from the app's
   directory (add `--exclude` for `.env` files and build caches), then deploy with
   `upload_id`. An app you just wrote in a few small files:
   pass them inline as `files: [{ path, content }]`. Never regenerate files that exist
   on disk as inline content. If the upload command is refused (a sandboxed shell or a
   proxy blocks `app.getspawnpoint.com`), say so and ask the user to allow that host for
   shell commands; only a bundle of a few small text files may fall back to inline.
   When the upload command cannot run at all (a sandboxed chat app), send small binary
   files such as images inline with `encoding: "base64"`, under about 3 MiB in total.
3. **Call `deploy_project`** with `name` (a short slug), `runtime` (`static`, `node`,
   `python`, or `docker`), and `files` or `upload_id`. Optional: `entrypoint`, `env`,
   `visibility`, `teardown_in`.
4. **Call `watch_deploy({ project_id })`** and relay each step it streams. It
   returns when the deploy settles or after `wait_seconds` (default 180). Still
   `spawning` then? Call it again: a deploy takes 1 to 3 minutes, sometimes up to 12.
   (No `watch_deploy`? Poll `get_project` every 15 seconds.)
5. **Hand over `url`** once the status is `running`, and say who can open it. Never
   surface `machine_url`. The console is <https://app.getspawnpoint.com/console>.

## Runtimes

- `static`: served as is. `node` and `python` must listen on port 8080 (`PORT` is set).
- Dependencies install on the machine from `package.json` or `requirements.txt` at the
  bundle root (`package-lock.json` gets `npm ci`). Never vendor them or ship
  `node_modules`.
- A `node` project with a `start` script runs `npm start` (and its `build` script
  first), so Next.js, Remix, Astro, Nuxt, SvelteKit, and Express deploy as themselves.
  Omit `entrypoint` then; otherwise it defaults to `index.js` / `main.py`.
- `docker` builds the root `Dockerfile` on the machine; the container listens on 8080.
  Use it only for compiled languages or system packages: it starts slower.

An app that serves HTML with no Open Graph tags gets `og:title`, `og:description`, and
`<meta name="twitter:card" content="summary">` in its first deploy, so the link unfurls
well in chat. Never replace tags the app already has.

## Who can open it

New projects are `restricted`: the owner and invited emails only. Pass
`visibility: "public"` when the user wants a link anyone can open, or flip it later with
`set_visibility`. `share_project({ project_id, email })` invites one person (an email
with the link, no account needed); `unshare_project` revokes.

## Secrets and config

Pass secrets in `env` or `set_env`, never in a file. `set_env({ project_id, set?,
unset? })` changes a live project's environment and restarts the app, no redeploy.

## Storage: where the app keeps its data

The app's own directory is replaced on every deploy, so never keep state there. Pick
by what the app stores:

| The app stores | Use | Call |
|---|---|---|
| A SQLite file, a JSON store, a small cache | the data directory, `SPAWNPOINT_DATA_DIR` (fallback `./data` locally) | none |
| Relational data, or the user asks for Postgres | managed Postgres, `DATABASE_URL` | `add_database({ project_id })` |
| User uploads, images, any file a browser sends or downloads | an S3 bucket, the `S3_*` variables | `add_storage({ project_id, public? })` |

- The data directory survives redeploys and sleep (`/home/ubuntu/data`, or `/data`
  inside a docker container; `get_project` reports it as `data_dir`). It lives on one
  machine and dies with the project.
- Call `add_database` after the first deploy: the app restarts with `DATABASE_URL`
  set, connects with any Postgres client, and creates its own tables. Idempotent; the
  database dies with the project. For a small app, SQLite in the data directory is
  the simpler answer.
- Each database has a size quota by plan and allows 20 connections. Past the quota it
  turns read-only (nothing is deleted): `get_project` shows `database.read_only` and a
  note. Relay the note; keep app connection pools small.
- Call `add_storage` after the first deploy: the app restarts with `S3_ENDPOINT`,
  `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, and `S3_SECRET_ACCESS_KEY` set. S3 SDKs
  do not read these names on their own: pass them to the client explicitly. The key
  reaches this one bucket. Idempotent; the bucket and its files die with the project.
- **Bytes never touch the machine.** The bucket is private by default: for an upload
  the app signs a presigned PUT URL and the browser sends the file straight to the
  bucket; for a download the app signs a presigned GET URL. Never stream uploads
  through the app. The bucket already accepts these browser requests (CORS).
- `public: true` (or `set_storage_visibility` later) only for files meant for
  everyone, such as a public gallery: anyone can read them at `S3_PUBLIC_URL/<key>`,
  outside the project's gate. Never for user documents. A public bucket serves files
  by exact key, never a listing, so the app lists with its own key. A restricted
  project with a public bucket gets a `warning`: relay it to the user.
- The app must still start before the `add_*` call: the first deploy succeeds only
  once the app serves a page. Read these variables when a request needs them, and
  answer with a clear message while one is missing instead of exiting; the call's
  restart brings the rest up.
- A public app with an owner-only action (an admin page, uploads only the owner
  makes) needs a token check: a secret in `env` the route (or the presign endpoint)
  compares against. Tell the owner the secret.
- A bucket key that was printed, logged, or committed: `rotate_storage_key` replaces it
  and the app keeps working; URLs signed with the old key stop working. `set_env`
  cannot change the `S3_*` variables.
- `remove_database` and `remove_storage` delete the data for good: ask the user first.

## When it goes wrong

- `error`: call `get_project_logs({ project_id })`, read the push log (the real pip or
  npm error is there), fix it, and redeploy under the same name. A redeploy updates in
  place and keeps the URL.
- Running but broken: `get_project_logs({ project_id, kind: "runtime" })` is the app's
  live log.
- Plan-limit error: the Free plan allows one live project. Offer to terminate one or to
  upgrade in the console.
- `sleeping`: idle, not down. Visiting the URL wakes it in about 30 seconds.
- Auth error: tell the user to run `/mcp`, select **spawnpoint**, and choose
  **Authenticate**. No spawnpoint server at all: run `/spawnpoint:setup-spawnpoint`.
- Inline calls are capped at 4 MiB, uploads at 20 MiB. Keep images web-sized (jpg or
  webp, under 200 KB each).

## Ending a project

- `terminate_project({ project_id, confirm: true })` destroys the machine and its data,
  the database and the bucket included. Ask the user first, in those words, and pass
  `confirm: true` only after they agree.
- Temporary or a demo? Pass `teardown_in` (`45m`, `2h`, `1d`) to `deploy_project`, or
  call `schedule_teardown({ project_id, in })` later.

## Other tools

`get_project` (status, health, machine, data directory, database, bucket),
`list_projects`, `get_project_visits` (who opened a shared link), `add_custom_domain` /
`verify_custom_domain` / `remove_custom_domain`, `whoami`. Every tool and argument:
<https://getspawnpoint.com/mcp.html>.
