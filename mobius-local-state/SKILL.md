---
name: mobius-local-state
description: Read the local state of Mobius with no change. Use it to find the active tasks, the task of one issue and its states, the cost of the agent turns, the copied issues and Workstreams, the Inbox items, the Lead events, the pending actions, the first prompt of a session, the release of the running Mobius, and the first release that has a merged pull request. Use it before you blame `main` for a behavior of the running Mobius.
metadata:
  roles: lead,coordinator,curator,researcher
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

- `tasks`: `id`, `repository`, `workstream_repository`, `issue`, `workstream`, `state`, `state_at`, `worker`, `branch`, `pull_request`, `review_rounds`, `fix_rounds`, `rerun_head`.
- `copied_issues`: `repository`, `workstream`, `number`, `repository_url`, `title`, `body`, `state`.
- `copied_workstreams`: `repository`, `number`, `title`, `body`, `autopilot`, `evergreen`.
- `sessions`: `id`, `role`, `repository`, `issue`, `workstream`, `started_at`, `ended_at`, `end_reason`.
- `transcript`: `id`, `session`, `time`, `kind`, `json`. The kind `prompt` has the prompt text in `$.text`.
- `task_states`: `id`, `task`, `state`, `role`, `session`, `started_at`, `ended_at`. Each row is one state of a task.
- `inbox_items`: `id`, `kind`, `repository`, `workstream_repository`, `workstream`, `issue`, `pull_request`, `time`, `dismissed_at`, `merging`, `merge_refusal`.
- `lead_events`: `id`, `repository`, `workstream`, `issue_repository`, `issue`, `kind`, `payload`, `time`, `delivered_at`, `held`, `held_at`.
- `events`: `id`, `time`, `repository`, `workstream`, `issue`, `actor`, `text`, `link`.
- `pending_actions`: `id`, `repository`, `workstream`, `issue`, `role`, `tool`, `arguments`, `state`, `error`, `result`, `time`, `decided_at`.
- `turn_usage`: `id`, `session`, `task`, `issue`, `workstream`, `repository`, `role`, `harness`, `model`, `effort`, `started_at`, `ended_at`, `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, `cost_usd`. Each row is one agent turn.
- `goose_db_version`: `version_id`, `tstamp`.

The column of the pull request is `pull_request`, not `pr`. When `state` is `working`, the column `worker` has the role of the Worker that runs on the task, for example `implementer` or `reviewer`. In each other state, `worker` keeps the role of the last Worker.

The role of the Lead in `sessions` and `turn_usage` is `lead_chat`, not `lead`. The other roles have their lower-case name, for example `implementer` or `reviewer`.

In `tasks`, `repository` is the repository of the task issue, and `workstream_repository` is the repository of its Workstream. In `copied_issues`, `repository` is the repository of the Workstream copy, and `repository_url` is the API URL of the repository of the issue. One Workstream can have issues with the same number in two repositories. In `lead_events`, `repository` is the repository of the Workstream, and `issue_repository` is the repository of the issue.

The times are RFC 3339 text in UTC, and the fraction of the seconds has a variable length, for example `2026-10-22T22:45:49.88685Z`. Thus a text compare gives the correct order only for times in different seconds. Give each bound of a time window with no fraction, for example `2026-10-22T22:00:00Z`.

## Queries

Active tasks:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select id, repository, issue, workstream_repository, workstream, state, pull_request, branch, review_rounds, fix_rounds from tasks where state <> 'ended' order by id"
```

Newest task of one issue. Always filter on `repository`, because two repositories can have an issue with the same number:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select id, state, state_at, worker, pull_request, fix_rounds, review_rounds from tasks where repository = '<owner>/<repo>' and issue = <number> order by id desc limit 1"
```

States of a task, newest first. Find the task id with the query above:

```sh
sqlite3 -readonly -header ~/.mobius/mobius.db "select state, role, session, started_at, ended_at from task_states where task = <id> order by id desc"
```

Lead events of a time window:

```sh
sqlite3 -readonly -header ~/.mobius/mobius.db "select id, repository, workstream, issue_repository, issue, kind, time, delivered_at, held_at, replace(substr(payload, 1, 200), char(10), ' ') from lead_events where time between '<from>' and '<to>' order by id"
```

Open issues of one Workstream:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select distinct repository_url, number, state, title from copied_issues where repository = '<owner>/<repo>' and workstream = <number> and state = 'open'"
```

Brief of a Workstream:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select number, title, autopilot, evergreen, body from copied_workstreams where repository = '<owner>/<repo>' and number = <number>"
```

Turns and cost of one role for each UTC day:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select date(started_at) d, count(*), round(sum(cost_usd), 2) from turn_usage where role = '<role>' and started_at >= '<day>' group by d"
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

4. Find the first release that has a merged pull request. Compare that tag with the tag of step 1.

   ```sh
   R=~/.mobius/repos/Mobius-Toolkit/Mobius.git
   git -C $R fetch -q origin --tags
   c=$(gh pr view <n> -R Mobius-Toolkit/Mobius --json mergeCommit --jq .mergeCommit.oid)
   test -n "$c" && git -C $R tag --contains "$c" | sort -V | head -1
   ```

   An empty result tells that the pull request is not merged, or that no release has it yet. The `test` is necessary: with no commit, `git tag --contains` reads the old local `HEAD` of the bare clone.
