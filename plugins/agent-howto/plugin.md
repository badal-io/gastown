+++
name = "agent-howto"
description = "Reference guide for agents bootstrapping a Gas Town demo with parallel polecats"
version = 1

[gate]
type = "manual"

[tracking]
labels = ["plugin:agent-howto", "category:docs"]
digest = false

[execution]
timeout = "0s"
notify_on_failure = false
severity = "low"
+++

# Agent Howto: Running a Gas Town Demo

A practical bootstrap guide for agents setting up a Gas Town demo that shows
multiple polecats working in parallel and visible in the convoy dashboard.

This document captures hard-won operational knowledge from actually running a
Gas Town demo end-to-end, including every failure mode discovered.

---

## Prerequisites

Gas Town requires four tools:

```bash
gt      # Gas Town CLI
bd      # Beads issue tracker
dolt    # Git-compatible SQL DB (backs beads)
tmux    # Terminal multiplexer (sessions run here)
```

After building `gt` from source, **always codesign before installing on macOS**:

```bash
make build
cp ./gt ~/.local/bin/gt
codesign --sign - --force ~/.local/bin/gt   # required — unsigned binary is SIGKILL'd
```

---

## Setup Flow

### 1. Create HQ (once per machine)

```bash
export GT_TOWN_ROOT=~/gt
gt install ~/gt --shell
gt git-init
```

### 2. Start the daemon

```bash
GT_TOWN_ROOT=~/gt gt daemon start
# Verify: GT_TOWN_ROOT=~/gt gt daemon status
```

### 3. Register a rig

```bash
GT_TOWN_ROOT=~/gt gt rig add <rig-name> <repo-url> \
    --local-repo <path-to-local-clone> \
    --prefix <2-3-letter-prefix> \
    --branch main

# Idempotent re-adoption (if directory already exists):
GT_TOWN_ROOT=~/gt gt rig add <rig-name> --adopt
```

### 4. Start witness and refinery

```bash
GT_TOWN_ROOT=~/gt gt rig start <rig-name>
# Verify: GT_TOWN_ROOT=~/gt gt rig status <rig-name>
```

---

## Spawning Parallel Polecats (The Right Way)

The key demo moment is showing 6+ polecats working in parallel, all visible in
the dashboard. There are two approaches — one correct, one that looks right but
silently fails.

### ❌ Wrong: Sling a convoy formula and expect separate sessions

```bash
# This spawns ONE polecat that runs convoy legs internally using the Agent
# tool — no separate tmux sessions appear in the dashboard.
gt sling mol-prd-review --var "problem=..." <rig> --create
```

The polecat picks up the convoy formula, reads its legs, and executes them
all as sub-agents within its own process. The dashboard shows 1 session.

### ✅ Correct: Batch sling individual beads

Create one bead per task, then batch-sling them. Each bead gets its own
polecat with its own tmux session:

```bash
export BEADS_DIR="$GT_TOWN_ROOT/$RIG_NAME/.beads"

# Create task beads
B1=$(bd create "Task: Requirements analysis" -d "Analyze requirements..." \
    --json 2>/dev/null | python3 -c "import json,sys; print(json.load(sys.stdin)['id'])")
B2=$(bd create "Task: Gap analysis"          -d "Identify gaps..." \
    --json 2>/dev/null | python3 -c "import json,sys; print(json.load(sys.stdin)['id'])")
# ... create B3-B6 similarly

# Batch sling — spawns one polecat per bead, all visible in dashboard
GT_TOWN_ROOT=~/gt gt sling $B1 $B2 $B3 $B4 $B5 $B6 <rig-name>
```

The batch sling creates one polecat per bead. All sessions appear in the
dashboard immediately.

---

## Demo Script Pattern

A production-quality demo script should be idempotent and support `--clean`.
See the reference implementation in the `badal-io/test-website` repo at
`gastown-demo.sh`. Key structure:

```bash
# Clean mode: nuke all rig state
do_clean() {
    GT_TOWN_ROOT="${TOWN_ROOT}" gt rig stop "${RIG_NAME}" --nuclear

    # CRITICAL: clear stale sling locks — interrupted runs leave .flock files
    # that cause "timed out acquiring assignee sling lock" on re-run
    find "${TOWN_ROOT}/.runtime/locks/sling" \
        -name "assignee_${RIG_NAME}_polecats_*.flock" -delete

    # Remove polecat worktrees from .repo.git (bare repo)
    REPO_GIT="${TOWN_ROOT}/${RIG_NAME}/.repo.git"
    git --git-dir="${REPO_GIT}" branch | grep "polecat/" \
        | while read -r branch; do
            git --git-dir="${REPO_GIT}" branch -D "$branch" 2>/dev/null || true
        done

    # Close open escalations so dashboard is clean
    BEADS_DIR="${TOWN_ROOT}/.beads" bd list \
        --label=gt:escalation --status=open --json 2>/dev/null \
        | python3 -c "import json,sys; print(' '.join(i['id'] for i in json.load(sys.stdin)))" \
        | xargs -r BEADS_DIR="${TOWN_ROOT}/.beads" bd close --force
}
```

---

## Dashboard

Launch with:

```bash
GT_TOWN_ROOT=~/gt gt dashboard --port 3300 --open
```

### Dashboard shows no sessions/polecats?

**This is a tmux socket mismatch bug.** Gas Town uses a custom tmux socket
(e.g., `gt-ff82fc`), but older builds of the dashboard call bare `tmux
list-sessions` which hits the default socket (which doesn't exist). The
dashboard renders empty worker/session panels even though all polecats are
running fine.

**Fix**: The bug is in `internal/web/fetcher.go` `runCmd()` and in
`internal/web/api.go` session preview. Both need to route tmux calls through
`tmux.BuildCommandContext()` which adds the `-L <socket>` flag.

This is fixed in the `fix/read-docs-repo-set` branch of this repo.

**Workaround** (if running unpatched): The beads data (convoys, work items,
hooks) will still render correctly. Only session/worker panels are blank.

---

## Running from Outside ~/gt

All `gt` commands that need to find the town root support `GT_TOWN_ROOT` as a
fallback when the current working directory is outside `~/gt`. Always set it:

```bash
export GT_TOWN_ROOT=~/gt
```

### Bug: `gt sling` / `gt formula run` fail from outside ~/gt

Affects: formula lookup, bead creation routing, `bd formula show`.

Root cause: Several internal functions used `workspace.FindFromCwd()` which
silently returns `("", nil)` when CWD is outside the town, instead of
`workspace.FindFromCwdOrError()` which falls back to `GT_TOWN_ROOT`.

Also: `bd` commands need both `GT_ROOT` (formula search path) AND `BEADS_DIR`
(database path) when invoked from outside `~/gt`. Pass both:

```go
BdCmd("formula", "show", formulaName).
    WithGTRoot(townRoot).
    WithBeadsDir(townBeadsDir)
```

Fixed in `fix/read-docs-repo-set`.

---

## Common Failure Modes

### Polecats won't spawn: "timed out acquiring assignee sling lock"

An interrupted run left a stale `.flock` file. Clear it:

```bash
rm ~/gt/.runtime/locks/sling/assignee_<rig>_polecats_<name>.flock
```

Add this to your `--clean` handler to prevent recurrence (see demo script above).

### Polecat is spawned but session never starts

`gt sling` output says "session start deferred" then fails silently. This means
the hook write failed (e.g., the sling lock timed out). Check:

```bash
GT_TOWN_ROOT=~/gt gt rig status <rig>   # look for "working" without a bead ref
```

Then manually sling the remaining beads.

### Deacon stuck escalations flood the dashboard

A stuck heartbeat escalation gets created every time the daemon detects the
deacon hasn't reported in. Close them before a demo:

```bash
BEADS_DIR=~/gt/.beads bd list --label=gt:escalation --status=open --json \
    | python3 -c "import json,sys; print(' '.join(i['id'] for i in json.load(sys.stdin)))" \
    | xargs BEADS_DIR=~/gt/.beads bd close --force
```

### `gt` binary gets SIGKILL'd after install on macOS

`make build` signs `./gt` but `cp ./gt ~/.local/bin/gt` strips the signature.
macOS kills unsigned binaries from non-standard locations:

```bash
# Always do this after cp:
codesign --sign - --force ~/.local/bin/gt
```

### Polecat resolves to `mol-prd-review` convoy but uses Agent tool not separate sessions

By design, a polecat that receives a convoy-type formula will use Claude's
built-in `Agent` tool for parallel legs — NOT separate tmux sessions. This
is not a bug in the formula; it's how Claude Code's agent parallelism works.

To force separate tmux sessions (required for dashboard visibility), use
batch sling with individual beads as shown above.

### Formula commands `--problem` / `--context` flags don't exist

Some formula templates (e.g., `mol-idea-to-plan`) contain instructions like:

```bash
gt formula run mol-prd-review --problem="..." --context="..."
```

This command **does not work**. `gt formula run` has no `--problem` flag.

The correct approach to sling a formula with variables is:

```bash
gt sling mol-prd-review --var "problem=..." --var "context=..." <rig> --create
```

### Dashboard shows issues from a different rig / can't find HQ convoys

The dashboard's `NewAPIHandler()` workDir must resolve to the town root, not
the CWD. Fixed in `fix/read-docs-repo-set` by using `workspace.FindFromCwdOrError()`.

---

## Rate-Limit Watchdog

If you don't have an Anthropic API key (e.g., using Gemini or a proxy), disable
the rate-limit watchdog plugin to prevent failed plugin runs from polluting the
dashboard:

```toml
# ~/gt/plugins/rate-limit-watchdog/plugin.md
[gate]
type = "manual"   # was: cooldown / duration = "3m"
```

---

## Key Commands Reference

```bash
# Town / HQ
gt daemon start|stop|status
gt dashboard --port 3300 --open

# Rig lifecycle
gt rig add <name> <url> --local-repo <path> --prefix <px>
gt rig start|stop <name>
gt rig status <name>
gt rig stop <name> --nuclear        # force-stop even with uncommitted work

# Work dispatch
gt sling <bead-or-formula> <rig>    # sling to existing polecat
gt sling <bead> <rig> --create      # spawn new polecat
gt sling $B1 $B2 $B3 <rig>         # batch sling → one polecat per bead
gt sling <formula> --var k=v <rig> --create  # formula with variables

# Beads
bd create "Title" -d "Description" --json
bd list --json
bd show <id>
bd close <id> --force
bd ready                             # unblocked work items

# Polecat workload
gt prime --hook                      # agent startup: what's on my hook?
gt mol status                        # molecule/formula progress
gt mail inbox                        # inter-agent messages
gt done                              # submit work to merge queue
```
