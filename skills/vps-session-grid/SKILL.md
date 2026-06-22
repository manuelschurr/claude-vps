---
name: vps-session-grid
description: Bring up or reattach a project's tiled tmux grid on the VPS — one Claude pane per worktree session. Trigger when the user wants to "bring up the grid for <project>", "open my sessions", "reattach the grid", "show me the worktree grid", or "rebuild my tmux panes". Calls the worktree-orchestrator engine's grid command; never reimplements tmux logic.
---

# VPS session grid

Bring up / reattach a project's tiled **tmux** grid — one `claude` pane per
worktree session — on the headless VPS. The engine's `grid` command does the
mechanics (reconcile + tile); this skill is the trigger.

    set -a; . ~/.orchestrator/env 2>/dev/null; set +a
    ORCH=~/code/worktree-dashboard/orchestrator.py
    cd ~/code/<project>
    python3 "$ORCH" grid

`grid` is **additive + safe**: it adds a pane for any worktree that lacks one,
tiles the layout, and never touches existing panes (won't kill a Claude you're
mid-conversation with). Re-run it anytime to reattach or pick up new worktrees.

## Steps
1. `cd ~/code/<project>` and run `python3 "$ORCH" grid`.
2. The agent can't attach you to tmux — relay the command it prints:
   `tmux attach -t <project>` (if you ran it yourself inside tmux, it already switched).

## Notes
- Needs `tmux` installed and at least one spawned worktree (spawn one first if none).
- One window, tiled, `claude` per worktree. The orchestration shell stays separate.
- Boundaries: never reimplement tmux/worktree logic — only via the engine `grid` command.
