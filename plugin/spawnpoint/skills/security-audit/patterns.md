# Credential patterns

These are the seven shapes spawnpoint's server refuses in a bundle
(`internal/deploy/scan.go`, `secretPatterns`), so the audit blocks exactly what
the server would refuse and nothing it would accept. Keep the two lists
identical: `internal/deploy/scan_patterns_test.go` fails when they drift.

| Kind | Pattern (Go RE2; for `grep -E`, replace `(?:` with `(`) |
|---|---|
| AWS access key id | `\b(?:AKIA\|ASIA)[0-9A-Z]{16}\b` |
| private key block | `-----BEGIN (?:RSA \|EC \|OPENSSH \|DSA \|PGP )?PRIVATE KEY-----` |
| Stripe secret key | `\b(?:sk\|rk)_live_[0-9A-Za-z]{16,}` |
| OpenAI or Anthropic API key | `\bsk-(?:ant-\|proj-)?[0-9A-Za-z_-]{20,}` |
| GitHub token | `\b(?:ghp\|gho\|ghs\|ghr)_[0-9A-Za-z]{36}\b\|\bgithub_pat_[0-9A-Za-z_]{22,}` |
| Google API key | `\bAIza[0-9A-Za-z_-]{35}\b` |
| Slack token | `\bxox[baprs]-[0-9A-Za-z-]{10,}` |

The `\|` in the table is an escaped pipe for markdown: the pattern itself has a
plain `|`.

One grep for all seven, from the app directory, skipping what never ships:

```bash
grep -rInE --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=.venv \
  --exclude-dir=venv --exclude='.env' --exclude='.env.*' \
  '\b(AKIA|ASIA)[0-9A-Z]{16}\b|-----BEGIN (RSA |EC |OPENSSH |DSA |PGP )?PRIVATE KEY-----|\b(sk|rk)_live_[0-9A-Za-z]{16,}|\bsk-(ant-|proj-)?[0-9A-Za-z_-]{20,}|\b(ghp|gho|ghs|ghr)_[0-9A-Za-z]{36}\b|\bgithub_pat_[0-9A-Za-z_]{22,}|\bAIza[0-9A-Za-z_-]{35}\b|\bxox[baprs]-[0-9A-Za-z-]{10,}' . \
  | sed -E 's/:([0-9]+):.*/:\1/'
```

The `sed` keeps the file and line and drops the matched text, so a key never
reaches the transcript twice.

Credential files by name, whatever they contain: `*.pem`, `*.key`, `*.p12`,
`*.pfx`, `id_rsa*`, `id_ed25519*`, `.npmrc` containing `_authToken`, `.pypirc`,
`.netrc`, `serviceAccount*.json`, `credentials.json`, `token.json`.
