---
name: vps-onboard-project
description: Onboard a project onto the VPS — import an existing GitHub repo or stand up a new one, configure it, and run its first worktree session at a public URL. Trigger when the user wants to "onboard project X", "add a project to the VPS", "import my repo onto the box", "set up a new project on the VPS", or "stand up <name>". Calls the worktree-orchestrator engine; never reimplements it; never reads secret values.
---

# Onboard a project

Policy layer that takes a project from "not on the box" to "running at a public
URL", for two flows: **import** an existing GitHub repo (deep), or stand up a
**new** project (high-level guidance). It calls the engine
(`~/code/worktree-dashboard/orchestrator.py`) for `init`/`spawn`, reuses the
`vps-dev-server` skill for the readiness-aware run + resource guard, and edits
on-box reverse-proxy config. The agent does the reasoning; it never owns secrets.

Setup (load the host's public-URL config; never hardcode the domain):

    set -a; . ~/.orchestrator/env 2>/dev/null; set +a
    ORCH=~/code/worktree-dashboard/orchestrator.py

## Hard rules
- **Never read or echo secret values** — not `.orchestrator/.secrets`, not `.env`.
  Determine which secrets are needed only from non-secret sources (code,
  `.env.example`, README, server config). Tell the user the keys; they fill them.
- **Skill = policy.** Never reimplement worktrees/ports/proxy — call the engine.
- **Confirm before impactful on-box changes** (reverse-proxy reload, spawning
  under memory pressure) and run the resource pre-check first (see vps-dev-server).
- Read public URLs from `status --json` (`servers[].url`) — never construct them.

## Does vs guides
- **Agent does (on-box, non-secret):** clone; inspect repo; draft `.orchestrator.toml`;
  run `init`; add the `*.<project>` reverse-proxy block + cert + reload; spawn +
  readiness; report URL.
- **Agent guides, user does (off-box / secret / console):** fill
  `.orchestrator/.secrets`; create the dev DB (e.g. Neon branch); set DNS / OAuth
  redirect URIs; choose dev vs prod.

## Prerequisites (check up front; stop if missing)
- `gh auth status` OK; engine present at `$ORCH`; the loopback proxy daemon
  running (`ss -ltn | grep 1337`); the reverse proxy (Caddy) active.
- The project's toolchain (e.g. `flutter`/`dart`, `node`) is on `PATH`. The engine
  runs servers with the launching shell's `PATH`, so put the toolchain in
  `~/.orchestrator/env` (which the setup block above sources). Verify `which <tool>`
  resolves before spawning, or the build fails with "command not found".

## Flow 1 — Import an existing GitHub repo
1. **Acquire** — `gh repo clone <owner/repo> ~/code/<name>`. Pick a **hostname-safe**
   `<name>` (lowercase, hyphens — no underscores or dots), because it becomes a label
   in the URL host. Confirm the name; skip the clone if `~/code/<name>` exists.
2. **Inspect** — read the repo to identify each server process (e.g. a web
   frontend, an API backend), its start command, working directory, port, and the
   env/secret KEYS it expects (from non-secret sources). Do NOT open secret files.
3. **Draft config** — run `python3 "$ORCH" init` (scaffolds `.orchestrator.toml`,
   `.orchestrator/.secrets`, `.gitignore`). Then write the drafted `[servers.*]`
   into `.orchestrator.toml`: `start_command`, `directory`, `[servers.*.env]` with
   `{port}`/`{url}` placeholders, `primary = true` on the web frontend. Show the
   draft; let the user tweak before writing.
4. **Secrets + external checklist** — list the secret keys needed for
   `.orchestrator/.secrets` and any external setup (dev DB, DNS, OAuth redirect
   URI). Ask the user to fill the secrets; wait for "done"; never look inside.

### Serve "main" (prototype — no worktrees)
For a prototype you can serve the project directly at the apex instead of spawning a
worktree session: run `python3 "$ORCH" spawn main` (runs the servers in `~/code/<project>`
in place, no worktree), wait for readiness, and report `https://<project>.<tld>`. The
generic `*.<tld>` block already serves it — no per-project Caddy work. Worktree sessions
can still be spawned later and coexist with `main`.

5. **On-box infra** — if `*.<project>.<tld>` isn't already served, add a
   reverse-proxy site block for it (wildcard cert via DNS-01). After editing the
   proxy config as root, **re-assert its ownership/permissions** (an edit can reset
   them so the proxy's service user can't read the file → reload fails), then
   **validate** the config and reload. Confirm before reloading. Prefer a shared
   snippet over copying the whole block per project.
6. **Spawn + run** — hand off to vps-dev-server: resource pre-check → `spawn <name>`
   the first session (the name, e.g. `1`, becomes the branch + URL label) →
   readiness wait → report the public URL (+ basic_auth reminder; + OAuth
   redirect-URI note if the app uses OAuth — each session is a distinct host).

## Flow 2 — New project (high-level guidance)
Guide the user to create the repo themselves — from their own template (e.g. a
flutter_bootstrap variant), then `gh repo create` + push. Do NOT build per-stack
scaffolding. Once the repo is on the box at `~/code/<name>`, continue at Flow 1
step 2.

## Idempotency
Re-runnable; resume where it left off: repo already cloned → skip; `.orchestrator.toml`
present → reuse/agree config, skip init; proxy block present → skip; secrets not
filled → pause for the user, resume at spawn.

## On error
- Missing prerequisite → say exactly what's missing, stop.
- Backend won't start with blank secrets → report it, point back to the checklist;
  don't spin.
- Never proceed past a secret/console step on assumption — wait for confirmation.
