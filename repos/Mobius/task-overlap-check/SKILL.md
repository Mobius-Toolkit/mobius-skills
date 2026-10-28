---
name: task-overlap-check
description: Find the work that a new task can collide with, before you write the task issue or the proposal of a task. Use it to choose a free migration version, and to find the open pull requests and the live task branches that change the same files.
metadata:
  roles: lead,researcher
---

# Overlap check for a new task

Do this check before you write a task issue or the proposal of a task that adds a migration or changes shared files. Two tasks with the same migration version, or with changes to the same lines, cause a conflict at the merge.

Mobius keeps a bare clone of each repository at `~/.mobius/repos/<owner>/<repo>.git`. The branch of a task has the name `mobius/<issue>` or `mobius/<issue>-<n>`. A branch can have commits before its pull request opens, and GitHub deletes the branch of a merged pull request. Mobius merges with a squash, so a merged branch is not an ancestor of `main`.

In Mobius-Toolkit/Mobius, the migrations are in `internal/store/migrations/`. The name of a migration file starts with a version of 14 digits, `YYYYMMDDhhmmss`. A version can have a date in the future.

## Free migration version

1. Read the newest versions on `main` and on all task branches. A version on a stale branch only makes the free version higher.

   ```sh
   R=~/.mobius/repos/<owner>/<repo>.git
   M=internal/store/migrations/
   git -C $R fetch -q origin
   git -C $R ls-tree --name-only origin/main $M | tail -3
   git -C $R for-each-ref --format='%(refname:short)' 'refs/heads/mobius/*' 'refs/remotes/origin/mobius/*' | while read b; do git -C $R ls-tree --name-only "$b" $M; done | sort -u | tail -3
   ```

2. Read the versions that the open issues keep. An open issue that names a version keeps that version for its task.

   ```sh
   gh issue list -R <owner>/<repo> --state open --limit 200 --json number,title,body | jq -r '.[] | (.body | [scan("20[0-9]{12}")] | unique | join(",")) as $v | select($v != "") | "\(.number) [\($v)] \(.title)"'
   ```

3. Choose one exact version that is greater than each version of steps 1 and 2. Write this version in the task issue or in the proposal. Do not write a range or "this version or higher".

## Open pull requests that change the same files

Set `<paths>` to a regular expression of the files of the new task, for example `web/src/Chat.tsx|internal/engine/copy.go`.

```sh
gh pr list -R <owner>/<repo> --state open --limit 200 --json number,title,files --jq '.[] | select(any(.files[].path; test("<paths>"))) | "\(.number) \(.title)"'
```

## Live task branches that change the same files

The branches of the live tasks are in the Mobius database. Give the files or the directories of the new task as `<files>`.

```sh
R=~/.mobius/repos/<owner>/<repo>.git
git -C $R fetch -q origin
sqlite3 -readonly ~/.mobius/mobius.db "select branch from tasks where state <> 'ended' and repository = '<owner>/<repo>' and branch <> ''" | while read b; do git -C $R diff --name-only "origin/main...$b" -- <files> 2>/dev/null | sed "s|^|$b: |"; done
```

## Result

When an open pull request or a live task branch changes the same code, name its issue in the Limits of the new task issue, or in the proposal. A Lead also tells the Owner. When the new task needs that work, also add its issue as a blocker.
