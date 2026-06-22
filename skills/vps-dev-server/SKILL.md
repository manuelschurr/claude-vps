---
name: vps-dev-server
description: Manage VPS dev servers for testing a worktree branch — start/restart a branch's server and return a public URL, and guard the VPS against running out of memory. Trigger when the user wants to "test/preview a branch", "show me a branch", "give me the URL", "restart the server", "stop/free the server", "what's running on the VPS", "free up memory", or "the VPS is full". Runs the worktree-orchestrator engine; never reimplements it.
---

# VPS Dev Server

Policy layer over the worktree-orchestrator engine. The engine
(`~/code/worktree-dashboard/orchestrator.py`) does the mechanics — worktrees,
ports, and a loopback Host-routing proxy on `127.0.0.1:1337` that a public-facing
reverse proxy fronts on `:443`. This skill decides *when* to act and reports the
public URLs the engine produces.

Run engine commands from the project repo root, loading the host's public-URL
config (domain/scheme/port live in a host-local file, never in this skill):

    set -a; . ~/.orchestrator/env 2>/dev/null; set +a
    ORCH=~/code/worktree-dashboard/orchestrator.py
    cd ~/code/<project>

Always get the actual public URLs from `status --json` (`servers[].url`) rather
than constructing them — the host's config determines the domain.

## Test / preview a branch
1. Resource pre-check (see Resource guard) BEFORE starting.
2. Start it: `python3 "$ORCH" restart <session>` (use `spawn` if it isn't registered yet).
3. Wait for REAL readiness — not just an open port. Poll `status --json` until the
   server is `up`. A Flutter-web dev server returns `index.html` (HTTP 200, ~1–2 KB)
   while `main.dart.js` is still compiling, so an open port is NOT readiness.
   Confirm the compiled bundle is actually served by hitting the loopback proxy
   directly (bypasses the public proxy / its auth), addressing the server by its
   registered Host — take the primary server's `url` from `status --json` and use
   its host:

       host=$(… primary servers[].url from status --json, scheme stripped …)
       curl -s -o /dev/null -w '%{size_download}\n' \
         -H "Host: $host" http://127.0.0.1:1337/main.dart.js

   Treat the frontend as ready only when this is large (> 100 KB / ~100000 bytes).
   While compiling it 404s or returns the small index shell. Poll every few
   seconds until it crosses the threshold (first compile can take a minute+).
4. Report the public URL from `status --json` (`servers[].url`, the `primary` one for
   the app). If the frontend sits behind the public proxy's auth, remind the user.

> **Auth gotcha — real OAuth per session.** URLs key off the *session id*, not the
> branch, so each session is a distinct backend host. If the app signs in with
> Google (or any OAuth), that session's redirect URI (the backend `url` from
> `status --json` + the app's callback path) must be whitelisted in the provider's
> console first, or login silently fails with a redirect-uri mismatch. When
> previewing a **new/different session** that needs auth, tell the user the exact
> redirect URI to add before they try to log in.

## Restart after a work session
When a worktree's task is done and the user wants to eyeball it, run `restart
<session>`, wait for readiness, and hand back the URL.

## Stop / free
`python3 "$ORCH" kill <session>` to stop servers and free RAM.

## Resource guard (run proactively at lifecycle moments)
- Before any spawn/restart, read `status --json`. If `memory.available_mb` < 1500,
  do NOT silently proceed: report the pressure and list running servers.
- Idle detection: a server is idle if `last_access` is older than 24h or null since
  `started_at`.
- Ask before killing: when memory is low or the user asks to "free up the VPS", list
  running servers with `rss_mb` + `last_access` and ASK which idle ones to kill. Never
  auto-kill without confirmation.
- Never silently OOM: if available memory is low and the user wants another server,
  surface it and propose reclaiming idle ones first.

## Boundaries
- Never edit ports/proxy/worktrees directly — only via the engine.
- Public exposure (reverse proxy / DNS / TLS) is one-time host infra, not this skill's job.
- Dev servers currently run as the same user as the engine — a future sandbox would
  isolate them under an unprivileged user.
