# The Claude Code plugin

Moved from the root CLAUDE.md; loads when work touches `plugin/`. The rules about shipping skill text with tool changes and keeping `patterns.md` in step with `scan.go` stay in the root file.

The public install source is **`spawnpoint-inc/spawnpoint-plugin`**, a separate public repo
carrying only the plugin bits; this app repo (`spawnpoint-inc/spawnpoint`) stays private. The
`publish-plugin` GitHub Action (`.github/workflows/publish-plugin.yml`) syncs it automatically
on every push to `main` that touches `plugin/` or `.claude-plugin/`, authenticated by the
`PLUGIN_DEPLOY_KEY` secret (a write deploy key on the plugin repo). The public repo's own
README and LICENSE live only there and are never overwritten by the sync.
