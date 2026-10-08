---
name: security-audit
description: Audit an app before deploying it to spawnpoint, and fix what is safe to fix. Finds secrets and files that should not ship, checks the runtime contract (PORT, bind address), flags debug mode, open CORS, unauthenticated admin, upload, or presign routes, private files in a public bucket, and credentials that reach the browser, and audits dependencies. Use before deploy_project, or when the user asks to check, audit, or review an app for security.
argument-hint: [app-directory]
allowed-tools: Bash(grep *) Bash(find *) Bash(npm audit *) Bash(npm install --package-lock-only *) Bash(uvx pip-audit *) Bash(docker build --check *) Bash(gitleaks dir *) Bash(osv-scanner *)
---

# Security audit before a spawnpoint deploy

Run it on the app directory (the argument, or the current directory) once the files
are final, before `create_upload` or `deploy_project`. Install nothing. End with one
table and a verdict.

## The rules

- **block**: no `deploy_project` until it is fixed or, where allowed, accepted.
  **warn**: fix it unless the user accepts. **note**: mention once.
- Only two findings can never be accepted: credential content and a private-key
  file. The server refuses those bundles anyway.
- **Fix what is safe and say so**: a port read from the environment, the bind
  address, a debug flag, a secret moved to `env`, an exclusion, and a token check on
  an admin, debug, or owner-only upload or presign route (generate the token, pass it
  as `env`, and give it to the owner once in your reply). Then deploy. **Ask first** only
  before changing what the app's own users get: narrowing CORS, removing a route,
  or making a route they use private.
- Never print a secret you found: file, line, and kind only.
- On a redeploy in the same session, rerun only if files changed. An acceptance holds
  for the session.

## 1. What ships

- [block] A credential-shaped string in any text file: the seven patterns in
  [patterns.md](patterns.md), with its one-line grep. Fix: move the value into `env`
  on `deploy_project` (or `set_env`), read it from the environment, and tell the
  user to **rotate the key**: it is in this transcript now. A spawnpoint bucket's
  key (the project's `S3_*` values) is rotated with `rotate_storage_key`.
- [block] A credential file by name (the list in patterns.md). Fix: exclude it.
  `.npmrc` matters twice: it ships, and `npm ci` on the machine reads it.
- [strip] `.env`, `.env.*`, `.git/`: exclude them. Name the variables `.env` held
  (never the values) and offer to pass each one as `env`.
- [exclude] `node_modules/`, `.venv/`, `venv/`, `__pycache__/`, `.next/`, `dist/` and
  `build/` when a build script exists, `.DS_Store`, `*.log`. These become the
  `tar --exclude` flags for `create_upload`'s command.
- [warn] Local databases and dumps (`*.sqlite`, `*.sqlite3`, `*.db`, `*.sql`,
  `*.dump`, `*.bak`): ask once whether a seeded demo is intended.
- [warn] Over about 15 MiB before compression: images first.

## 2. The runtime contract

spawnpoint sets `PORT` (8080, or 8081 behind the access gate when the project is
restricted) and the gate proxies to `127.0.0.1`. A hardcoded port breaks restricted
mode; a `127.0.0.1` bind breaks public mode. Both are one-line fixes: make them.

- node: `listen()` with a literal port and no `process.env.PORT`. [warn]
- python: `app.run()` or `uvicorn.run()` without `host="0.0.0.0"`, or `uvicorn
  main:app` without `--host 0.0.0.0`; a port literal without `os.environ["PORT"]`.
  [warn] Fix: `host="0.0.0.0", port=int(os.environ.get("PORT", 8080))`.
- docker: no `EXPOSE 8080` and no `8080` or `$PORT` in `CMD`/`ENTRYPOINT`. [warn]
- static: a directory without `index.html` is listed, every file in it visible;
  symlinks are followed. [warn]

## 3. Framework mistakes

- [block] Flask `debug=True` or `FLASK_DEBUG=1`, Django `DEBUG = True`: the debugger
  runs arbitrary code from the browser.
- [block] A `NEXT_PUBLIC_*`, `VITE_*`, or `REACT_APP_*` name holding a secret, in a
  file or in `env`: those are built into the browser bundle.
- [block] Docker `ARG` or `ENV` named like a secret with a literal value. Run `docker
  build --check .` when docker is on PATH.
- [warn] A literal `secret_key`/`SECRET_KEY`, or Django's `django-insecure-` prefix:
  sessions become forgeable. Fix: generate one and pass it as `env`.
- [warn] CORS allowing every origin with credentials (`cors({ origin: '*',
  credentials: true })`, `origin: true`, reflecting the request origin,
  `CORSMiddleware(allow_origins=["*"], allow_credentials=True)`, `flask_cors` with
  `supports_credentials=True` and default origins).
- [warn] `express.static(__dirname)` or `express.static('.')`: serves the source.
- [warn] Django `ALLOWED_HOSTS = []` with `DEBUG = False`: every request fails.
- [block if public, warn if restricted] An unauthenticated admin or write route: a
  path like `/admin`, `/api/admin`, `/debug`, `/internal`, or any POST, PUT, PATCH, or
  DELETE (uploads included) with no auth middleware or check. Admin, debug, and
  owner-only routes get the token check (above). A write route the app's users are
  meant to call (posting a comment, adding a note) is a note, not a block: say it is
  open to anyone with the link.

## 4. Data and storage

- [block] A database URL, an `S3_*` key, or any other credential in client-side code
  or a public build variable: the browser gets it. The browser gets presigned URLs
  the server signs, never the key.
- [block if public, warn if restricted] An upload or presign endpoint anyone can
  call: strangers fill the disk or the bucket, and the owner pays. Owner-only uploads
  get the token check (above); uploads the app's users make get a size and content
  type cap in what the app accepts or signs.
- [warn] User documents in a public bucket (`add_storage` or
  `set_storage_visibility` with `public: true`): anyone with a file's URL reads it,
  and a restricted project's gate does not cover the bucket. Keep it private and
  serve files through presigned GET URLs.
- [warn] State kept inside the app's directory: every redeploy replaces it. Move it
  to the data directory (`SPAWNPOINT_DATA_DIR`).
- [warn] SQL built from request input by string formatting: use the driver's
  parameters. Show the line.
- [warn] A connection string or key written to logs.

## 5. Dependencies

The runtime's own auditor, only when present: `npm audit --omit=dev
--audit-level=high` (with no lockfile, first `npm install --package-lock-only
--ignore-scripts`), `uvx pip-audit -r requirements.txt`. `gitleaks dir --redact` and
`osv-scanner` only when already on PATH. Advisories never block: a high or critical
with a fix is a warn with the upgrade command, the rest is a note. A missing tool is
one line in the table, not a stop.

## 6. Exposure

Projects deploy `restricted` by default. If the audit found an unauthenticated
admin, write, upload, or presign route and the user wants a public link, recommend
adding auth first, or staying restricted and inviting people with `share_project`.
Restricted covers the app, not a public bucket: when a storage tool or
`set_visibility` returns a `warning`, relay it.

## Output

```
| severity | finding | done or to do |
| block | config.js:12 Stripe secret key | moved to env STRIPE_SECRET_KEY; rotate the key |
| warn | main.py:88 app.run() binds 127.0.0.1 | fixed: host="0.0.0.0", port from PORT |
| block | server.js:41 POST /admin/reset has no auth | fixed: requires ADMIN_TOKEN (passed as env, given to you below) |
| note | npm audit: no high or critical | none |

verdict: no blocks left; deploying.
```
