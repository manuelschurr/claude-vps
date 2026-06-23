# claude-vps

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin for running Claude Code on a VPS across a project's whole lifecycle — set up a fresh box, onboard projects and their worktrees, test branches from any device by clicking a public URL, and keep the box from running out of memory.

It's a **policy layer**: the skills decide *when* to act and report results; the mechanics (worktrees, ports, the Host-routing reverse proxy, tmux) belong to the [worktree-dashboard](https://github.com/manuelschurr/worktree-dashboard) engine (`orchestrator.py`), which the skills call and never reimplement.

> Personal tool. It assumes a specific host setup (the engine plus a reverse proxy exposing dev servers over HTTPS). `vps-bootstrap` reproduces that setup. Sharing it because the pattern may be useful.

## Status

The plugin ships **four skills**, covering the lifecycle end to end:

- **`vps-bootstrap`** — set up a fresh VPS to run claude-vps: install packages (tmux, mosh, fail2ban, Caddy), clone the engine, configure swap, firewall, the reverse-proxy service, and host config — guiding you through secrets/DNS/domain.
- **`vps-onboard-project`** — onboard a project onto the box: import an existing GitHub repo or stand up a new one, draft its server config, guide secrets/external setup, and run its first worktree session at a public URL.
- **`vps-dev-server`** — start/restart a worktree branch's dev server, wait for *real* readiness, and hand back a public URL; plus a memory/idle **resource guard** that warns before spawning under pressure and asks before killing idle servers.
- **`vps-session-grid`** — bring up or reattach a project's tiled tmux grid (one Claude pane per worktree session); additive and safe to re-run.

## Install

Via my [plugin marketplace](https://github.com/manuelschurr/c200v-marketplace):

```
/plugin marketplace add manuelschurr/c200v-marketplace
/plugin install claude-vps@c200v-marketplace
```

## Skills

### `vps-bootstrap` — *"bootstrap this box"*
One-time setup of a fresh box. Does the on-box, non-secret work (packages, Caddy + the cloudflare DNS module, swap, `ufw` + fail2ban, `~/.orchestrator/env`, the reverse-proxy systemd unit, cloning the engine) and **guides** you through the secret/external pieces (domain + Cloudflare DNS + API token, extra SSH keys, git identity). Idempotent; confirms before risky system changes.

### `vps-onboard-project` — *"onboard `<owner/repo>`"*
Takes a project from "not on the box" to a live URL. **Import** an existing repo (deep: clone → inspect → draft `.orchestrator.toml` → guide secrets/external → add the project's reverse-proxy block → spawn → URL) or stand up a **new** one (high-level guidance, then the same back half). Never reads secret values.

### `vps-dev-server` — *"test this branch"*, *"free up memory"*
The daily driver. **Test/preview a branch** (resource pre-check → start/restart → wait for **real** readiness, not just an open port → return the public URL), **restart after a work session**, **stop/free** to reclaim RAM, and a **resource guard** that reads `status --json`, warns on low memory, flags idle servers, and asks before killing anything.

### `vps-session-grid` — *"bring up the grid for `<project>`"*
A per-project tiled **tmux** cockpit — one `claude` pane per worktree. Additive reconcile (adds only missing panes, orders them by worktree, never disturbs a running Claude), then prints the attach command. Rebuilds your whole grid after a reboot in one shot.

## How it fits together

```
internet (PC / phone)                 the host (single VPS)
       │ https                ┌──────────────────────────────────┐
       ▼                      │  Caddy :443  (wildcard TLS cert,   │
     Caddy ─────────────────► │  optional basic_auth on the FE)    │
       │  http, Host kept     │            │                       │
       ▼                      │            ▼                       │
 orchestrator proxy 127.0.0.1:1337 ──► dev servers (per worktree)  │
       │ routes by Host       └──────────────────────────────────┘
       ▼
 the skills  ← policy: call the engine, report the public URL
```

### Prerequisites

- The [worktree-dashboard](https://github.com/manuelschurr/worktree-dashboard) engine present at `~/code/worktree-dashboard/orchestrator.py` (the skills shell out to it) — `vps-bootstrap` clones it.
- A reverse proxy (Caddy) fronting the loopback proxy with a wildcard TLS cert; the proxy itself runs as a systemd service (`orchestrator-proxy`).
- The host's public-URL config in `~/.orchestrator/env` (`ORCH_TLD`, `ORCH_SCHEME`, `ORCH_URL_PORT`) — the skills source it, so the domain lives on the host, not in the plugin.

## Shared principles

- **Skill = policy** — never reimplement worktrees/ports/proxy/tmux; call the engine.
- **Never reads secret values** — identifies the *keys* you need; you fill `.orchestrator/.secrets` / `/etc/caddy/.env`. The agent only appends user-pasted **public** SSH keys.
- **No domain in the plugin** — sources `~/.orchestrator/env`; reads URLs from `status --json`.
- **Idempotent + confirm-before-risky** — re-runnable; impactful system changes are confirmed first.

## Known limitations

- **URLs use the session id, not the branch name.**
- **Real auth per worktree** — apps using OAuth must have each session's redirect URI whitelisted (each session is a distinct backend host).
- **Dev servers run as the same user as the engine (root)** for now — a future sandbox would isolate them under an unprivileged user.

## License

[MIT](LICENSE) © 2026 Manuel Schurr
