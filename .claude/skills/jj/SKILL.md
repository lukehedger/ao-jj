---
name: jj
description: >
  Operate jujutsu (jj) repos efficiently. Use whenever the working dir
  contains a .jj/ directory, the user mentions jj/jujutsu, or asks for
  version-control operations in a jj repo. Covers the mental-model
  differences from git, command translations, multi-workspace patterns
  for parallel agents, and recovery via the op log.
---

# Operating jj repos

## Detect first

Before running any `git` command, check for `.jj/`:

```sh
test -d .jj && echo jj || echo git
```

If `.jj/` exists, prefer `jj` commands. A colocated repo (`.jj/` AND `.git/`) is common — jj is the source of truth; running `git` mutators (commit, branch, reset) will desync state. Read-only git commands (`git log`, `git diff`, `git show`) are fine and sometimes faster.

## Mental model — what's different from git

- **Working copy is a commit.** Every `jj` invocation snapshots the working dir into the change at `@`. There is no staging area and no "uncommitted changes" — only a commit you keep editing.
- **Change ID vs commit ID.** A *change* has a stable ID (letters, e.g. `kxqz...`) that survives rebases/amends. A *commit* has a hash (digits/letters) that changes when you rewrite. Refer to work by change ID.
- **`@` = current change.** `@-` is its parent. `@+` is its child. Use these as revsets.
- **Rewriting is normal.** `squash`, `rebase`, `abandon`, `describe` are everyday ops, not surgery.
- **Conflicts live in commits.** A rebase never aborts midway; it produces commits with conflict markers that you resolve later.
- **Bookmarks ≠ branches.** Bookmarks are named pointers you move explicitly. They do not auto-follow new commits the way git branches do. Push with `jj git push -b <name>`.
- **Op log.** Every repo mutation is recorded. `jj op log`, `jj op restore <id>` is your undo.

## Command cheatsheet (git → jj)

| Intent | git | jj |
|---|---|---|
| Show state | `git status` | `jj st` |
| Show history | `git log --oneline` | `jj log` |
| Show diff | `git diff` | `jj diff` |
| Show a commit | `git show <sha>` | `jj show <rev>` |
| Start new work | `git checkout -b foo` | `jj new -m "foo"` |
| Edit message | `git commit --amend` | `jj describe -m "..."` |
| Stage & commit some changes | `git add -p && git commit` | `jj split` (interactive) or `jj squash -i` |
| Move changes into parent | n/a | `jj squash` |
| Switch to a change | `git checkout <sha>` | `jj edit <rev>` (mutable) or `jj new <rev>` (build on top) |
| Drop a commit | `git reset --hard HEAD~` | `jj abandon <rev>` |
| Rebase onto main | `git rebase main` | `jj rebase -d main` |
| Pull | `git pull` | `jj git fetch` then `jj rebase -d main@origin` |
| Push branch | `git push -u origin foo` | `jj bookmark set foo -r @ && jj git push -b foo` |
| Undo last op | `git reflog` + reset | `jj op log` then `jj op restore <id>` |
| Resolve conflicts | edit + `git add` + continue | edit the conflicted commit; `jj resolve` for guided |

## Workflows

### Single change, iterating

```sh
jj new -m "fix auth race"   # start
# ...edit files...
jj diff                     # review
jj describe -m "fix auth race in middleware"  # refine message
jj new                      # start a fresh child change when this one is done
```

You never run `commit`. The snapshot happens implicitly on every `jj` invocation.

### Splitting accidental work

If two unrelated edits landed in `@`:

```sh
jj split           # interactive selector splits @ into two commits
# or
jj split <files>   # split out specific paths
```

### Pushing to a remote

```sh
jj git fetch
jj rebase -d 'main@origin'
jj bookmark set my-feature -r @
jj git push -b my-feature
```

To update a PR after edits: rebase if needed, then `jj git push -b my-feature` again. jj will refuse if the remote moved unexpectedly — investigate, do not force.

## Multi-agent / parallel work

Several agents may be running in their own jj workspaces against this same repo backend. If `jj workspace list` shows more than one workspace, or the user mentions parallel agents / isolating work, **read [MULTI_AGENT.md](MULTI_AGENT.md)** for conventions, orientation commands, and the stay-in-your-lane rules.

## Recovery

`jj op log` shows every operation. To undo:

```sh
jj op log              # find the operation id you want to roll back to
jj op restore <op-id>  # restore repo to that state
```

This rewinds refs, bookmarks, the working-copy commit — everything. It does NOT undo pushes to remotes.

If the working copy got an unwanted snapshot (debug print, scratch file), either:

- `jj abandon` the bad change (if no children depend on it), or
- `jj op restore` to before the snapshot.

## Pitfalls to avoid

- **Don't run `git commit`, `git checkout -b`, `git reset` in a colocated repo.** Use the jj equivalents. Read-only git is fine.
- **Don't assume the working copy is "uncommitted".** It's a real commit; secrets and debug files in it become history. `jj diff` before describing.
- **Don't rewrite commits another agent is building on** without coordinating — change IDs survive but their parents may move out from under collaborators.
- **Don't push without setting a bookmark first.** `jj git push` alone won't push an anonymous change.
- **Conflicted commits are valid.** Don't try to "abort" a rebase; resolve the conflict commit or `jj op restore` to before the rebase.
- **`jj edit` makes a commit mutable.** That's usually what you want for fixing the current change, but it means the working copy now points at history. Use `jj new <rev>` if you want to build on top instead.

## Quick diagnostic

When confused about state:

```sh
jj log -r 'all()' --limit 20   # recent commits across all branches
jj st                          # what's in @
jj op log --limit 5            # what just happened
```

Almost any mistake is recoverable via `jj op restore`. Lean on it.
