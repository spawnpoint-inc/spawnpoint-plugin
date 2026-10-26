---
name: deploy-to-spawnpoint
description: Put an app, site, or page online with spawnpoint and get a link to share, and get the user signed in to spawnpoint. Use when the user asks to put something online, publish, host, deploy, ship, or launch it, or for a link other people can open, whether or not they name spawnpoint (unless they name another host); to set up, connect, or sign in to spawnpoint; or when a spawnpoint tool answers that it is not signed in.
---

# Deploy to spawnpoint

spawnpoint runs a small app on its own machine and returns a share link that survives
redeploys.

These skills are spawnpoint plugin version 0.16.4. When `whoami` reports a newer
`plugin_version`, tell the user once that their spawnpoint plugin is out of date: in
claude.ai or the desktop app, **Customize**, **Plugins**, **spawnpoint**, **Update**; in
Claude Code, `/plugin`, **Marketplaces**, **spawnpoint**, update. When the user asks
which version they have, this line is the answer.

## Signing in

This plugin carries the spawnpoint server, so there is nothing to install or register
from here. Call `whoami`: when it answers, the user is connected. When it answers 401 or
says there is no grant, give the user the steps for where they are, in one reply, and
never ask for a token or key (there is none to copy):

- **Claude Code** (a terminal, an IDE, or the desktop app's Code tab): restart Claude
  Code if the plugin was just installed, run `/mcp`, select **spawnpoint** (listed as
  `plugin:spawnpoint:spawnpoint`), choose **Authenticate**, sign in with an email in
  the browser, and click **Approve**. Also tell them once to run `/plugin`, open
  **Marketplaces**, select **spawnpoint**, and turn on **auto-update**: third-party
  marketplaces do not update on their own, and the skills change often.
- **A chat or Cowork** (claude.ai, the desktop app, mobile): open **Customize**, then
  **Plugins**, select **spawnpoint**, and open its **Connectors** tab. Installing the
  plugin does not add its connector: when the tab says **Not added**, add it first.
  Then click **Connect**, sign in, and click **Approve**, and start a new chat if the
  tools still do not appear. Never run an installer or `claude mcp add` from a chat's sandbox: that sets
  up the sandbox, not the user's Claude.

No spawnpoint tools at all in Claude Code means the plugin's server did not load: run
`curl -fsSL https://getspawnpoint.com/install | bash`, then restart Claude Code.
No spawnpoint tools in a chat means its connector is not connected or is off in this
chat. Say so before you build anything, since the app has to be built again in a new
chat: give the chat steps above, and say to check that spawnpoint is on under **+**,
then **Connectors**, in the new chat.

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
   `visibility`, `teardown_in`, `inference` (open models, below).
4. **Call `watch_deploy({ project_id })`** and relay each step it streams. It
   returns when the deploy settles or after `wait_seconds` (default 180). Still
   `spawning` then? Call it again: a deploy takes 1 to 3 minutes, sometimes up to 12.
   A static site is `running` at once.
   (No `watch_deploy`? Poll `get_project` every 15 seconds.)
5. **Hand over `url`** once the status is `running`, and say who can open it. Never
   surface `machine_url`. The console is <https://app.getspawnpoint.com/console>.

## Runtimes

- `static`: served by spawnpoint from storage: live in seconds, no machine, never asleep.
  `node` and `python` must listen on port 8080 (`PORT` is set).
- Dependencies install on the machine from `package.json` or `requirements.txt` at the
  bundle root (`package-lock.json` gets `npm ci`). Never vendor them or ship
  `node_modules`.
- A `node` project with a `start` script runs `npm start` (and its `build` script
  first), so Next.js, Remix, Astro, Nuxt, SvelteKit, and Express deploy as themselves.
  Omit `entrypoint` then; otherwise node defaults to `package.json` `main`, else
  `server.js`, else `index.js`, and python to `main.py`.
- `docker` builds the root `Dockerfile` on the machine; the container listens on 8080.
  Use it only for compiled languages or system packages: it starts slower.

An app that serves HTML with no Open Graph tags gets `og:title`, `og:description`, and
`<meta name="twitter:card" content="summary">` in its first deploy, so the link unfurls
well in chat. Never replace tags the app already has.

## Who can open it

New projects are `restricted`: the owner and invited emails only (on a team, every
member of the team too; see Teams). Pass
`visibility: "public"` when the user wants a link anyone can open, or flip it later with
`set_visibility`. `share_project({ project_id, email })` invites one person (an email
with the link, no account needed), or everyone at a company when `email` is its domain,
`@company.com` (no invite goes out: give the user the link to pass along). Consumer
domains like `@gmail.com` are refused. `unshare_project` revokes either.

When the user names the address they want, such as lunch-orders.spwt.app, pass
`slug: "lunch-orders"` to `deploy_project`, or call `set_share_slug` on an existing
project. The old address keeps working for 30 days. Do not pick an address the user did
not ask for: a generated one is private by default.

## Teams

When `whoami` shows a `team`, the user signed this agent in for that team: every app
it deploys belongs to the team (`deploy_project` answers with the team's name), and
the `plan` and limits `whoami` reports are the team's. Say so when you hand over a link.

- Every member of the team can open the team's apps by signing in with their own
  email. Nothing needs sharing; `share_project` is for people outside the team.
- A member cannot delete an app another member built, or its database or bucket. The
  refusal names who can (the member who built it, or a team admin): relay it.
- To deploy to a team instead of their own account, or back, the user disconnects
  this agent under **Connected agents** in the console and signs it in again (see
  "Signing in"); the approval page asks where its apps go, under **Deploy as**. No
  tool argument picks the team.
- Members, roles, and the team's plan live on the team's page in the console (the
  **Team** tab), and only a team admin changes them. No tool does.
- The team space is one place for the team's files (priorities, customer lists,
  briefs) that every member and every team app reads. `list_team_space` shows what is
  there and `read_team_space` reads a file; check it when a task needs the team's
  context. `upload_to_team_space` puts files from disk into it. A team app reads the
  space from its `SPAWNPOINT_TEAM_S3_*` variables (a read-only key), which it gets on
  its first deploy after the space has a file.
- Team skills: the server's instructions list the team's skills by name. When the
  user's task matches one, call `get_skill` and follow its `SKILL.md`. To save one for
  the team, `publish_skill` takes a folder with a `SKILL.md` (front matter `name` and
  `description`) the way `deploy_project` takes an app. A skill another member
  published can be replaced only by them or by a team admin; if the call refuses, tell
  the user who published it rather than publishing under a new name.

## MCP servers

An MCP server is a `node` or `python` app: Streamable HTTP at `/mcp` on port 8080,
stateless (Node SDK: `sessionIdGenerator: undefined`; Python: `FastMCP(..., host="0.0.0.0",
stateless_http=True)` with `mcp<2` pinned, since 2.x renamed FastMCP). Who may call it
sets the visibility:

- The user and the people they share it with: leave it `restricted` (the default). The
  gate signs MCP clients in through spawnpoint's own sign-in, the one Claude Code already
  has, and admits the owner and shared emails. The app needs no auth of its own, and must
  not check an `Authorization` header: the gate owns that header on a restricted project,
  so a token of the app's own could never reach it. Give the user
  `claude mcp add --transport http <name> <url>/mcp` (in claude.ai or the desktop app:
  **Settings**, **Connectors**, **Add custom connector** with `<url>/mcp`); the client
  asks them to sign in.
- Anyone with the link: deploy it `public`, guard it with a generated token in `env`,
  checked as `Authorization: Bearer`, and give the user
  `claude mcp add --transport http <name> <url>/mcp --header "Authorization: Bearer <token>"`.

## Secrets and config

Pass secrets in `env` or `set_env`, never in a file. `set_env({ project_id, set?,
unset? })` changes a live project's environment and restarts the app, no redeploy.

A `.env` file never ships, and nothing on the machine is named `.env`: the app gets its
variables as its environment.

- Put every variable from the app's `.env` in `deploy_project`'s `env`, with its value
  from that file (skip `PORT` and `NODE_ENV`: spawnpoint sets them). Never make up a
  replacement value, and never repeat a value or any part of one in your reply.
- `dotenv.config()` and `load_dotenv()` keep working: with no file, the app reads its
  environment. Code that reads `.env` as a file (`fs.readFileSync('.env')`,
  `open('.env')`, `dotenv_values()`) must read `process.env` or `os.environ` instead.
- A loader that stops when the file is missing (Go's `godotenv.Load()` followed by
  `log.Fatal`, and the like): read the environment, or ignore its missing-file error.
- Never `COPY` or `ADD` a `.env` in a Dockerfile: the deploy is refused.
- A `deploy_project` answer whose `note` names variables: the app lacks them. Pass
  them with `set_env`.

## Model providers

An app that calls OpenAI or Anthropic (the SDK, the Vercel AI SDK, LangChain, or raw
HTTP) can run on open models through spawnpoint with no provider key: read the
`open-models` skill before deploying it. It asks the user once, picks the model, and
deploys with `inference: true`. An app you write that needs a model: use the open
model from the start, no question needed.

## Storage: where the app keeps its data

The app's own directory is replaced on every deploy, so never keep state there. Pick
by what the app stores:

| The app stores | Use | Call |
|---|---|---|
| A SQLite file, a JSON store, a small cache | the data directory, `SPAWNPOINT_DATA_DIR` (fallback `./data` locally) | none |
| Relational data, or the user asks for Postgres | managed Postgres, `DATABASE_URL` | `add_database({ project_id })` |
| User uploads, images, any file a browser sends or downloads | an S3 bucket, the `S3_*` variables | `add_storage({ project_id, public? })` |
| Files already on disk the site links to: images, videos, installers, anything over the bundle cap | the same bucket, public | `add_storage({ project_id, public: true })`, then `upload_to_storage({ project_id, files })` |

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
- Files you have on disk that the site or app needs (a gallery's photos, a video, an
  installer, anything that would push the bundle over 20 MiB): `upload_to_storage` with
  the paths, relative to the folder you run its `command` from; each path is the file's
  key. Run the command with bash from that folder, then link to each file's `url`
  (public bucket). No server is involved and the key never leaves spawnpoint. At most 50
  files per call; refused while the bucket is over its plan's limit. This works for a
  `static` project too, which is the usual case.
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
- A managed `DATABASE_URL` that was printed, logged, or committed:
  `rotate_database_password` gives the database a new password and the app keeps
  working; the tables and data stay. `set_env` cannot change it either.
- `remove_database` and `remove_storage` delete the data for good: ask the user first.

## Sleep and always on

An app on a machine sleeps after an hour with no visitors, and the next visit waits about
30 seconds. To keep one up, call `set_always_on({ project_id, always_on: true })`; a
static site never sleeps. The plan sets how many apps can be always on: `whoami`
shows the `plan`, its `subscription` (`active`, `complimentary`, or `none`), and
`always_on_used` against `always_on_limit`. Relay a refusal as given, and at the limit
ask the user which app to turn off rather than turning one off yourself.

## When it goes wrong

- `error`: call `get_project_logs({ project_id })`, read the push log (the real pip or
  npm error is there), fix it, and redeploy under the same name. A redeploy updates in
  place and keeps the URL.
- Running but broken: `get_project_logs({ project_id, kind: "runtime" })` is the app's
  live log.
- Plan-limit error: the Free plan allows one live project. Offer to terminate one, and
  relay the error, which links the page listing what each plan includes. On a team,
  terminating makes no room, so do not offer it: the team's plan is a team admin's to
  change. A team holds apps once a team admin with Pro covers it, on the team's page.
- `sleeping`: idle, not down. Visiting the URL wakes it in about 30 seconds; to stop it
  sleeping, see Sleep and always on.
- Asleep and "cannot start" (the computer its machine lives on is full): waiting is
  safe, and visitors see a page saying so. To move it now, ask the user, then redeploy
  with `move_machine: true`; files the app kept in its data directory are deleted with
  the old machine, while the bucket, database, env, and URL stay.
- `transfer.over` on a static site: it served its plan's monthly data and shows a
  "taking a break" page until the 1st. Tell the user; upgrading lifts it at once.
- Not signed in (a 401, or no grant): give the steps under "Signing in".
- Inline calls are capped at 4 MiB, uploads at 20 MiB (50 MiB for a static site). Keep images web-sized (jpg or
  webp, under 200 KB each), and put big files in a bucket with `upload_to_storage`
  instead of the bundle.

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
