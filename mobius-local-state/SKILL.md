---
name: mobius-local-state
description: Read the local state of Mobius with no change. Use it to find the active tasks, the copied issues and Workstreams, the first prompt of a session, and the release of the running Mobius. Use it before you blame `main` for a behavior of the running Mobius.
---

# Local state of Mobius

Mobius keeps its state in the SQLite file `~/.mobius/mobius.db`. The binary of the running Mobius is `~/.mobius/app/mobius`.

WARNING: Always open the database with `sqlite3 -readonly`. Do not write to the database. Mobius uses it while it runs.

## Columns

Read the columns of a table before you write a query:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select name from pragma_table_info('tasks')"
```

These columns are the most useful:

- `tasks`: `id`, `repository`, `workstream_repository`, `issue`, `workstream`, `state`, `state_at`, `branch`, `pull_request`, `review_rounds`, `fix_rounds`, `rerun_head`.
- `copied_issues`: `repository`, `workstream`, `number`, `repository_url`, `title`, `body`, `state`.
- `copied_workstreams`: `repository`, `number`, `title`, `body`, `autopilot`, `evergreen`.
- `sessions`: `id`, `role`, `repository`, `issue`, `workstream`, `started_at`, `ended_at`, `end_reason`.
- `transcript`: `id`, `session`, `time`, `kind`, `json`. The kind `prompt` has the prompt text in `$.text`.
- `goose_db_version`: `version_id`, `tstamp`.

The column of the pull request is `pull_request`, not `pr`.

In `tasks`, `repository` is the repository of the task issue, and `workstream_repository` is the repository of its Workstream. In `copied_issues`, `repository` is the repository of the Workstream copy, and `repository_url` is the API URL of the repository of the issue. One Workstream can have issues with the same number in two repositories.

## Queries

Active tasks:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select id, repository, issue, workstream_repository, workstream, state, pull_request, branch, review_rounds, fix_rounds from tasks where state <> 'ended' order by id"
```

Open issues of one Workstream:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select distinct repository_url, number, state, title from copied_issues where repository = '<owner>/<repo>' and workstream = <number> and state = 'open'"
```

Brief of a Workstream:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select number, title, autopilot, evergreen, body from copied_workstreams where repository = '<owner>/<repo>' and number = <number>"
```

First prompt of a session:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select json_extract(json, '$.text') from transcript where session = <id> and kind = 'prompt' order by id limit 1"
```

## Release of the running Mobius

The running Mobius can be older than `main`. Do these steps before you blame `main` for a behavior:

1. Read the release tag of the running binary. The tag is in the `-ldflags` line, for example `Release=v0.1.188`. A local build has no tag.

   ```sh
   go version -m ~/.mobius/app/mobius | grep -- -ldflags
   ```

2. Read the time of the upgrade to that release, in UTC. Data from before this time can show a defect that the running release already fixed.

   ```sh
   TZ=UTC stat -f '%Sc' -t '%FT%TZ' ~/.mobius/app/mobius
   ```

3. Find if a fix commit is in the running release. The result `behind` or `identical` tells that the release has the commit. The result `ahead` or `diverged` tells that the release does not have it.

   ```sh
   gh api repos/Mobius-Toolkit/Mobius/compare/<tag>...<commit> --jq .status
   ```
