# Multi-agent / parallel work in jj

Read this when the user has told you that you are one of several parallel agents working on this repo, or mentions "parallel agents", "isolate work", "workspaces", or running multiple Claude Code instances against the same repo.

The pattern: **you create your own jj workspace, work entirely inside it, and stop.** The user runs other Claude Code instances doing the same against the same repo backend. The user integrates afterwards.

## Setup — create your workspace before doing anything else

1. **See what exists** so you pick a non-colliding name:

   ```sh
   jj workspace list
   ```

2. **Pick a slug** for your task — short, kebab-case, descriptive (e.g. `auth-fix`, `pagination`, `parser-rewrite`). Don't number them; names collide less than numbers do.

3. **Create the workspace** as a sibling directory of the main checkout:

   ```sh
   REPO=$(basename "$(jj workspace root)")
   SLUG=<your-task-slug>
   jj workspace add "../${REPO}-wt-${SLUG}"
   ```

4. **Move into it.** The Bash tool's cwd persists across calls:

   ```sh
   cd "../${REPO}-wt-${SLUG}"
   pwd && jj workspace root   # confirm you're in the new workspace
   ```

5. **Start a fresh change off the same base the user is on.** Default to `trunk()` unless told otherwise:

   ```sh
   jj new 'trunk()' -m "agent: <one-line task summary>"
   ```

6. **Optionally set a bookmark** so the user has a stable handle:

   ```sh
   jj bookmark set "agent-${SLUG}" -r @
   ```

From here, all your file edits and `jj` commands happen inside this workspace. Do not `cd` back to the main checkout.

## Stay in your lane

The repo backend is shared. Commits and op-log entries from every workspace are visible from every workspace.

- **Only mutate `@` and ancestors you created.** Other workspaces' `@` commits show up in `jj log` annotated with their workspace name. Never `jj edit`, `jj abandon`, or `jj rebase` them.
- **Don't touch bookmarks you didn't create.** Even ones that look stale may be in use.
- **Don't `jj git push`.** The user integrates. If you think a push is needed, stop and ask.
- **Don't `jj op restore` past operations from other workspaces.** Op log is shared — restoring rolls back *everyone's* state. Use `jj abandon` on your own change instead.
- **Coordinate on shared files via the change graph, not the filesystem.** If your task overlaps another agent's, expect a conflicted commit at integration time; don't try to read or pre-merge their working copy.

## Snapshot hygiene matters more here

Every `jj` invocation snapshots your working copy into `@`, and that snapshot is immediately visible to other workspaces via the shared backend. Keep scratch files, debug prints, and secrets out of the workspace, or `jj abandon` before stepping away.

## When stuck

- `jj op log --limit 20` shows recent ops across *all* workspaces — useful for spotting whether your confusion comes from another agent's action.
- If your `@` got reparented unexpectedly, another agent likely rebased a shared ancestor. `jj log -r 'ancestors(@)'` to see what moved.
- If you got confused about which workspace you're in, `jj workspace root` is authoritative.

## Finishing

Leave a single described change at `@`. Summarize what you did, including:

- the workspace path you created
- the bookmark name (if you set one)
- the change ID at `@`

Then stop. Do not delete your workspace, do not push, do not rebase onto main. The user integrates and tears down.
