---
name: bare-clone-read
description: Read the files of `main` or of an open pull request from the bare clone of Mobius, with no worktree and no checkout, and the diff of a pull request with no generated files. Use it when you have no worktree of the repository, when your worktree is not at the newest `main`, or at the review of a pull request.
metadata:
  roles: lead
---

# Read code from the bare clone

Mobius keeps a bare clone of each repository at `~/.mobius/repos/<owner>/<repo>.git`. Its fetch refspec is `+refs/heads/*:refs/remotes/origin/*`, so `git fetch origin` updates `origin/main`.

WARNING: Do not check out, commit, or push in the bare clone. Do not add a worktree to it. Mobius adds, removes and prunes the worktrees of the tasks there. Only fetch and read.

Each Bash call has a new shell. Set `R` again in each call:

```sh
R=~/.mobius/repos/<owner>/<repo>.git
git -C $R fetch -q origin
git -C $R log --oneline -3 origin/main
```

Fetch before each read. A clone with no fetch can be older than `main`.

## Files of `main`

List the files of a directory:

```sh
git -C $R ls-tree --name-only origin/main <directory>/
```

Show one file, or a range of its lines:

```sh
git -C $R show origin/main:<path>
git -C $R show origin/main:<path> | sed -n <first>,<last>p
```

Search the files. Put the paths after `--`, in quotes, so that the shell does not expand them. In a `git grep` path, `*` also matches `/`: `'internal/*.go'` matches the Go files of each directory below `internal`.

```sh
git -C $R grep -n -E '<pattern>' origin/main -- '<paths>'
```

Each line of the result starts with `origin/main:<path>:<line>:`.

## Files on disk

A tool that reads files on disk, for example `grep -r` or `wc`, needs a copy of the tree. Extract the tree into a new temporary directory. To copy only some directories, give them after `origin/main`.

```sh
d=$(mktemp -d)
git -C $R archive origin/main <directories> | tar -x -C "$d"
echo "$d"
```

Remove the copy with `rm -rf "$d"` after the read.

## Files of an open pull request

GitHub keeps the head of each pull request at `refs/pull/<n>/head`. The fetch refspec does not get it. Fetch it into the local ref `refs/remotes/pr/<n>`. The `+` lets the fetch replace the ref after a force push.

```sh
git -C $R fetch -q origin "+pull/<n>/head:refs/remotes/pr/<n>"
git -C $R log --format='%h %s' origin/main..refs/remotes/pr/<n>
git -C $R diff --stat origin/main...refs/remotes/pr/<n>
git -C $R diff origin/main...refs/remotes/pr/<n> -- '<paths>'
git -C $R show refs/remotes/pr/<n>:<path>
```

Write the full ref name. The clone can also have an old ref `refs/pr/<n>`, and git reads it first for the short name `pr/<n>`.

Use three dots in `diff` to see only the changes of the pull request. Two dots compare the two tips, so the result also has the changes of `main` after the merge base, in reverse.

Find the commits of `main` that the pull request does not have:

```sh
git -C $R log --format='%h %cI %s' refs/remotes/pr/<n>..origin/main
```

Fetch the ref again after a new push to the pull request. The old ref keeps the old head.

## Diff of a pull request with no generated files

In Mobius-Toolkit/Mobius, `make generate` writes these files. Do not review them line by line.

| Generated file | Source |
| --- | --- |
| `internal/store/db.go`, `internal/store/models.go`, `internal/store/*.sql.go` | `internal/store/queries.sql` and `internal/store/migrations/` (sqlc) |
| `api/openapi.json` | the handlers in `internal/api` |
| `web/src/api/api.gen.ts` | `api/openapi.json` (orval) |

`.mobius/check` runs `make generate` and fails when a generated file is not current. Thus a head that passed the check has current generated files.

Fetch the ref of the pull request as above. Then read the diff of the source files with no generated files and no tests, and then the diff of the tests:

```sh
git -C $R diff origin/main...refs/remotes/pr/<n> -- . ':!internal/store/db.go' ':!internal/store/models.go' ':!internal/store/*.sql.go' ':!api/openapi.json' ':!web/src/api/api.gen.ts' ':!*_test.go' ':!web/e2e'
git -C $R diff origin/main...refs/remotes/pr/<n> -- '*_test.go' web/e2e
```

When the output is too long, give the directories one at a time, for example `-- internal/engine ':!*_test.go'`.

To read only the commits after a head that you read before, put `<old head> refs/remotes/pr/<n>` in place of `origin/main...refs/remotes/pr/<n>`.
