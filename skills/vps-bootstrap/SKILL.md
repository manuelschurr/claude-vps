---
name: vps-bootstrap
description: Set up a fresh VPS to run claude-vps — install packages (tmux, mosh, fail2ban, Caddy), clone the engine, configure swap, firewall, the reverse-proxy service, and host config; guide secrets/DNS/domain. Trigger when the user wants to "bootstrap this box", "set up a fresh VPS", or "set up the host for claude-vps". Does the on-box non-secret setup (confirming risky steps); never reads secret values.
---

# Bootstrap a VPS for claude-vps

One-time setup of a fresh box to run claude-vps. Policy + agent reasoning: the agent
does the on-box, non-secret steps (confirming the risky ones) and guides the user
through secrets/DNS/domain. The fresh-box counterpart to `vps-onboard-project`.

**Prerequisite:** Claude Code + the `claude-vps` plugin are already installed (that's
how this skill is running). Everything else is below.

## Hard rules
- **Never read or echo secret values** (the Cloudflare API token, etc.). The user
  places secrets; the agent only appends user-pasted **public** SSH keys.
- **Confirm before impactful system changes** — enabling ufw (allow the SSH port
  FIRST), adding swap, systemd changes.
- **Idempotent:** check each step's state and skip if already done. Safe to re-run.

## Does vs guides
- **Agent does (on-box):** apt packages; Caddy + cloudflare module; clone the engine;
  write env / CLAUDE.md / Caddyfile / systemd units; swap; ufw + fail2ban; the proxy service.
- **Agent guides, user does:** domain + Cloudflare zone + `*.<domain>` A record → this
  box's IP; the scoped API token → `/etc/caddy/.env`; extra SSH public keys; git identity.

## Sequence (idempotent — skip any step already satisfied)

### A. Guidance / user provides
1. Print the box's public IP (`curl -s -4 ifconfig.me`) and tell the user to create, in
   Cloudflare for their domain, an `*.<domain>` A record -> that IP (DNS-only / grey cloud).
2. Have them create a scoped Cloudflare API token (Edit zone DNS) and write it to
   `/etc/caddy/.env` as `CF_API_TOKEN=...` (you may `mkdir -p /etc/caddy`; never read the value).
3. Ask for any extra SSH public keys (phone / other laptop); append each to
   `~/.ssh/authorized_keys` only if not already present.
4. Ask for git identity; run `git config --global user.name` / `user.email`.

### B. Packages
`apt-get install -y tmux mosh fail2ban`. Install Caddy with the Cloudflare DNS module:
add the Caddy apt repo (keyring + sources list per caddyserver.com), `apt-get install -y
caddy`, then `caddy add-package github.com/caddy-dns/cloudflare`.

### C. Engine
If `~/code/worktree-dashboard` is absent, clone it:
`git clone https://github.com/manuelschurr/worktree-dashboard ~/code/worktree-dashboard`.

### D. Host config (confirm swap + ufw)
- `~/.orchestrator/env` (chmod 600): `ORCH_TLD=<domain>`, `ORCH_SCHEME=https`,
  `ORCH_URL_PORT=` (empty), plus a `PATH=<toolchain-bin>:$PATH` line if a toolchain is installed.
- `~/.claude/CLAUDE.md`: a short memory-awareness note using detected RAM / cores.
- **Swap** (confirm): if `swapon --show` is empty, create a swapfile (~= RAM size),
  `mkswap` / `swapon`, and add `/swapfile none swap sw 0 0` to `/etc/fstab`.
- **ufw** (confirm): detect the SSH port (`ss -ltnp | grep sshd`), `ufw allow <ssh>/tcp`
  FIRST, then `ufw allow 80,443/tcp` and `ufw allow 60000:61000/udp` (mosh), then
  `ufw --force enable`. Enable fail2ban: `systemctl enable --now fail2ban`.

### E. Caddy
- Ensure `/etc/caddy/.env` is owned `root:caddy` mode `640` (token already placed by the user).
- systemd drop-in `/etc/systemd/system/caddy.service.d/override.conf`:
  `EnvironmentFile=/etc/caddy/.env` and an `ExecStart=` without `--environ`
  (clear then re-set: `ExecStart=` then `ExecStart=/usr/bin/caddy run --config /etc/caddy/Caddyfile`).
- Minimal `/etc/caddy/Caddyfile` (a global options block with the user's email only —
  `vps-onboard-project` adds per-project `*.<project>.<domain>` site blocks).
  `chown root:caddy /etc/caddy/Caddyfile && chmod 640`.
- `systemctl daemon-reload`; `systemctl enable --now caddy`.

### F. Reverse-proxy service
Write `/etc/systemd/system/orchestrator-proxy.service`:

    [Unit]
    Description=worktree-orchestrator reverse proxy
    After=network.target

    [Service]
    ExecStart=/usr/bin/python3 /root/code/worktree-dashboard/orchestrator.py proxy -p 1337
    Restart=always
    RestartSec=2

    [Install]
    WantedBy=multi-user.target

Stop any detached proxy already on :1337, then `systemctl daemon-reload` and
`systemctl enable --now orchestrator-proxy`. (Replaces the detached process; survives reboot.)

### G. Verify
Caddy + orchestrator-proxy + ufw + fail2ban all active & enabled; `ss -ltn` shows
:1337, :80, :443; `swapon --show` non-empty; `~/.orchestrator/env` + `~/.claude/CLAUDE.md`
present; `~/code/worktree-dashboard` cloned; `mosh-server` installed; pasted keys in
`authorized_keys`. A live public URL needs a project — hand off to `vps-onboard-project`.

## On error
- Missing domain / token the user owns -> state exactly what's needed, pause, resume when ready.
- Never enable ufw without the SSH-allow rule first (anti-lockout).
- Never proceed past a secret / console step on assumption — wait for confirmation.
