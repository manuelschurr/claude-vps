# claude-vps — working notes

A Claude Code plugin for running Claude Code on a VPS across a project's whole
lifecycle. Four skills, all thin **policy** layers over the worktree-orchestrator
**engine** (`~/code/worktree-dashboard/orchestrator.py`):

| Skill | Does |
|---|---|
| `vps-bootstrap` | set up a fresh box (packages, Caddy, swap, firewall, the reverse-proxy systemd service, host config) |
| `vps-onboard-project` | clone/scaffold a project, configure it, run its first worktree session at a public URL |
| `vps-dev-server` | start/restart a branch's dev server (readiness-aware) + a memory/idle resource guard |
| `vps-session-grid` | bring up/reattach a project's tiled tmux grid (one `claude` pane per worktree) |

## Shared principles (hold across all four skills)

- **Skill = policy.** Never reimplement worktrees/ports/proxy/tmux — call the engine
  (`orchestrator.py`) or another skill. Mechanics live in the engine.
- **Never read or echo secret values.** Identify needed secret *keys* from non-secret
  sources only; the user fills `.orchestrator/.secrets` / `/etc/caddy/.env`. The agent
  only appends user-pasted **public** SSH keys.
- **No domain in the plugin.** Source `~/.orchestrator/env` for the public-URL config;
  read actual URLs from `status --json` (`servers[].url`) — never construct them, never
  hardcode the domain.
- **Idempotent + confirm-before-risky.** Each skill skips already-done steps; confirm
  before impactful changes (ufw, swap, systemd, proxy reload, spawning under memory
  pressure).

## The engine

`orchestrator.py` is stdlib-only at runtime. Commands the skills use: `init`, `spawn`,
`restart`, `kill`, `status --json`, `grid`, `proxy`. It is kept in sync across two
repos — the canonical dev copy `worktree-dashboard/` (where the pytest tests live) and
the `worktree-orchestrator` skill copy (`scripts/orchestrator.py`). **Engine changes are
TDD'd in the canonical copy, then `cp`'d to the skill copy; both are pushed.** Pure
helpers (e.g. `panes_to_create`, `worktree_paths_under`, `reorder_swaps`,
`parse_free_mb`) are unit-tested; live tmux/process behavior is validated by running it.

## On the VPS

The host runs Caddy (per-project `*.<project>.<domain>` wildcard certs via DNS-01,
token in `/etc/caddy/.env`), the orchestrator reverse proxy as a systemd service
(`orchestrator-proxy`, survives reboot), `ufw` (22/80/443 + mosh `60000:61000/udp`),
and swap. `vps-bootstrap` sets all of this up; `vps-onboard-project` adds each project's
Caddy site block.

## Editing skills

Each `SKILL.md` is markdown policy — keep it domain-free (no host domain/IP/passwords),
source `~/.orchestrator/env`, and read URLs from `status --json`. After editing, the
skills are symlinked into `~/.claude/skills/` on the box. `specs/` and `plans/` are
local design docs, git-ignored — not published.
