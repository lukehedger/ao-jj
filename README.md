# ao-jj

agent orchestrator + jujutsu

- one jujutsu workspace per agent, each on its own anonymous change, with a supervisor (human or another agent) reviewing via `jj log`/`jj diff` and using `jj squash`/`jj rebase` to integrate
- jj skill (Claude is used to git!)
- default [`ao`](https://github.com/lukehedger/ao) instruction to use `jj` workspaces to isolate work

![Demo](https://github.com/user-attachments/assets/a3dd6c50-0498-42bd-bd85-fc1433ba9305)

![jj example](https://github.com/user-attachments/assets/07ab1411-4aca-4d59-b6b0-0fff3b703222)

![jj example](https://github.com/user-attachments/assets/6174b571-5777-4797-8426-33896f6af1a4)

## Why jututsu?

Several properties of [jujutsu](https://jj-vcs.dev) (`jj`) make it better than git for multi-agent workflows:

- **Workspaces** (`jj workspace add`) - each agent gets its own working copy backed by the same repo. Lighter than git worktrees; commits made in one workspace are immediately visible in the others, so agents can hand work off without pushing.

- **Auto-snapshotting working copy** - every jj invocation snapshots the working dir as a commit. Agents can't "forget to commit", partial work is always recoverable and you can diff what an agent did at any point without it explicitly staging.

- **Operation log + `jj op restore`** - every repo mutation is logged. If an autonomous agent corrupts history (bad rebase, wrong squash), you rewind the operation, not just the files. A real safety net for letting agents run unattended.

- **First-class conflicts** - conflicts live inside commits instead of blocking the index. An agent doing a rebase or merge never gets stuck in an "aborted, fix and continue" state; it can produce a conflicted commit and keep moving and a human (or another agent) resolves later.

- **Stable change IDs across rebases** - an agent's change keeps its identity through history rewrites, so coordinating "agent A's work" across reorderings is tractable.

- **Concurrent-safe ops** - two agents running jj against the same repo simultaneously produce divergent operations that jj reconciles, rather than corrupting refs.

## Supervising parallel agents

Commands to run from any workspace while parallel Claude Code agents are working in their own jj workspaces. The repo backend is shared, so it doesn't matter which workspace you run them from.

### Overview

```sh
jj workspace list             # all workspaces and their @ commits
jj log                        # all visible commits; each workspace's @ is marked with @<name>
jj log -r 'visible_heads()'   # one line per active head - quick "where is everyone"
```

### Inspect agent's work

```sh
jj show <bookmark|change>                       # message + full diff for a change
jj diff -r <bookmark>                           # just the diff
jj diff --summary -r <bookmark>                 # files-touched list, no diff body
jj log -r 'ancestors(<bm>) & ~ancestors(main)'  # the agent's full chain off main
```

### Compare two agents' work

```sh
jj diff --from agent-a --to agent-b                  # what differs between them
jj log -r 'agent-a | agent-b'                        # both chains side by side
jj log -r '(agent-a..agent-b) | (agent-b..agent-a)'  # commits unique to each
```

### Detect issues

```sh
jj log -r 'conflicts()'              # any commit with unresolved conflicts
jj log -r 'divergent()'              # same change ID with multiple versions (two agents touched the same thing)
jj log -r 'description(exact:"")'    # changes with no description (agent forgot to describe)
jj log -r 'empty()'                  # empty changes (agent abandoned without cleanup)
```

### Audit the op log

```sh
jj op log --limit 30                 # recent operations across all workspaces
jj op log -p --limit 10              # same, with the diff each op produced
jj op show <op-id>                   # full detail of one operation
jj op log -T 'id.short() ++ " " ++ time.start() ++ " " ++ description'  # compact
```

Op log entries are tagged with the workspace that produced them - useful for tracing "which agent did this?".

### Roll back a bad op

```sh
jj op log --limit 20    # find the op-id just before the damage
jj op restore <op-id>   # rewind everything to that point
```

This affects all workspaces. To drop one agent's bad change surgically, prefer `jj abandon <change>` from inside that workspace.

### Tear down a finished workspace

```sh
jj workspace forget <name>           # detach from the repo
rm -rf <workspace-dir>               # remove the directory
jj bookmark delete <agent-bookmark>  # if you used bookmarks
```
