# claude-vps

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin for running Claude Code on a VPS — managing projects and their worktrees on a remote box, and testing branches from any device by clicking a public URL, without letting the box run out of memory.

It's a **policy layer**: the skills decide *when* to act and report results; the mechanics (worktrees, ports, the Host-routing reverse proxy) belong to the [worktree-orchestrator](https://github.com/manuelschurr/worktree-orchestrator) engine, which the skills call and never reimplement.

> Personal tool, evolving. It assumes a specific host setup (the orchestrator engine plus a reverse proxy exposing dev servers over HTTPS). Sharing it because the pattern may be useful.

## Status

Today the plugin ships three skills:

- **`vps-dev-server`** — start/restart a worktree branch's dev server, wait for *real* readiness, and hand back a public URL to test it from any device; plus a memory/idle **resource guard** that warns before spawning under pressure and asks before killing idle servers.
- **`vps-onboard-project`** — onboard a project onto the box: import an existing GitHub repo or stand up a new one, draft its server config, guide secrets/external setup, and run its first worktree session at a public URL.
- **`vps-session-grid`** — bring up or reattach a project's tiled tmux grid (one Claude pane per worktree session); additive and safe to re-run.

Planned next (each its own skill): packaging the one-time host **bootstrap**.

## Install

Via my [plugin marketplace](https://github.com/manuelschurr/c200v-marketplace):

```
/plugin marketplace add manuelschurr/c200v-marketplace
/plugin install claude-vps@c200v-marketplace
```

## `vps-dev-server`

Trigger it by asking things like *"test this branch"*, *"give me the URL"*, *"restart the server"*, *"what's running on the VPS"*, or *"free up memory"*.

- **Test / preview a branch** — resource pre-check → start/restart the branch's servers via the engine → wait for **real** readiness (not just an open port; for a Flutter-web frontend it confirms the compiled `main.dart.js` is actually served, since the dev server returns the small `index.html` shell while still compiling) → hand back the public URL.
- **Restart after a work session** — re-start, wait for ready, return the URL.
- **Stop / free** — kill a session's servers to reclaim RAM.
- **Resource guard** — reads the engine's `status --json`; warns on low available memory before spawning, flags idle servers (by last access), and **asks before killing** anything. Never silently OOMs the box.

## How it fits together

```
internet (PC / phone)                 the host (single VPS)
       │ https                ┌──────────────────────────────────┐
       ▼                      │  reverse proxy :443  (wildcard TLS │
  reverse proxy ───────────►  │  cert, optional auth on the FE)    │
       │  http, Host kept     │            │                       │
       ▼                      │            ▼                       │
 orchestrator proxy 127.0.0.1:1337 ──► dev servers (per worktree)  │
       │ routes by Host       └──────────────────────────────────┘
       ▼
 skill  ← policy: calls the engine, reports the public URL
```

### Prerequisites

- The [worktree-orchestrator](https://github.com/manuelschurr/worktree-orchestrator) engine present at `~/code/worktree-dashboard/orchestrator.py` (the skills shell out to it).
- A reverse proxy fronting the loopback proxy with a wildcard TLS cert (frontend auth optional).
- The host's public-URL config in `~/.orchestrator/env` (`ORCH_TLD`, `ORCH_SCHEME`, `ORCH_URL_PORT`) — the skill sources it, so the domain lives on the host, not in the plugin.

## Known limitations

- **URLs use the session id, not the branch name.**
- **Real auth per worktree** — apps using OAuth must have each session's redirect URI whitelisted (each session is a distinct backend host).
- **Dev servers run as the same user as the engine** for now — a future sandbox would isolate them under an unprivileged user.

## License

[MIT](LICENSE) © 2026 Manuel Schurr
