---
name: mobius-local-state
description: Read the local state of Mobius with no change. Use it to find the active tasks, the task of one issue and its states, the cost of the agent turns, the usage limit windows of Claude Code, the issues and Workstreams, the Inbox items, the Lead events, the pending actions, the first prompt of a session, the release of the running Mobius, the first release that has a merged pull request, and the server log with the poll errors of one repository. Use it before you blame `main` for a behavior of the running Mobius.
metadata:
  roles: lead,coordinator
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
- `issues`: `id`, `repository`, `number`, `title`, `body`, `state`, `author`, `workstream`, `parent`, `kind`, `triage`. `workstream` is the `id` of a row of `workstreams`. The `kind` is `task` or `project`.
- `workstreams`: `id`, `repository`, `number`, `title`, `body`, `state`, `autopilot`, `auto_merge`, `evergreen`.
- `sessions`: `id`, `role`, `repository`, `issue`, `workstream`, `started_at`, `ended_at`, `end_reason`.
- `transcript`: `id`, `session`, `time`, `kind`, `json`. The kind `prompt` has the prompt text in `$.text`.
- `task_states`: `id`, `task`, `state`, `role`, `session`, `started_at`, `ended_at`. Each row is one state of a task.
- `inbox_items`: `id`, `kind`, `repository`, `workstream_repository`, `workstream`, `issue`, `pull_request`, `time`, `dismissed_at`, `merging`, `merge_refusal`.
- `lead_events`: `id`, `repository`, `workstream`, `issue_repository`, `issue`, `kind`, `payload`, `time`, `delivered_at`, `held`, `held_at`.
- `pending_actions`: `id`, `repository`, `workstream`, `issue`, `role`, `tool`, `arguments`, `state`, `error`, `result`, `time`, `decided_at`.
- `turn_usage`: `id`, `session`, `task`, `issue`, `workstream`, `repository`, `role`, `harness`, `model`, `effort`, `started_at`, `ended_at`, `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, `cost_usd`. Each row is one agent turn.
- `goose_db_version`: `version_id`, `tstamp`.

The column of the pull request is `pull_request`, not `pr`. When `state` is `working`, the column `worker` has the role of the Worker that runs on the task, for example `implementer` or `reviewer`. In each other state, `worker` keeps the role of the last Worker.

The role of the Lead in `sessions` and `turn_usage` is `lead_chat`, not `lead`. The other roles have their lower-case name, for example `implementer` or `reviewer`.

In `tasks`, `repository` is the repository of the task issue, and `workstream_repository` is the repository of its Workstream. In `issues`, `repository` is the repository of the issue, and `workstreams.repository` is the repository of its Workstream. One Workstream can have issues with the same number in two repositories. In `lead_events`, `repository` is the repository of the Workstream, and `issue_repository` is the repository of the issue.

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
sqlite3 -readonly ~/.mobius/mobius.db "select i.repository, i.number, i.state, i.title from issues i join workstreams w on w.id = i.workstream where w.repository = '<owner>/<repo>' and w.number = <number> and i.state = 'open'"
```

Brief of a Workstream:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select number, title, autopilot, auto_merge, evergreen, body from workstreams where repository = '<owner>/<repo>' and number = <number>"
```

Turns and cost of one role for each UTC day:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select date(started_at) d, count(*), round(sum(cost_usd), 2) from turn_usage where role = '<role>' and started_at >= '<day>' group by d"
```

Windows of the 5-hour usage limit of Claude Code. `s` is the start of the window, and `top` is the highest usage that Claude reported in the window (1 is the limit). `top_at` is the time of the first report of `top`. `usd` is the cost that Mobius recorded in the window. Mobius records the cost of a turn at its end, so a turn that still runs is not in `usd`. `top` can also include usage of the account outside Mobius:

```sh
sqlite3 -readonly -header -column ~/.mobius/mobius.db "
with u as (select strftime('%Y-%m-%dT%H:%M:%SZ', time) time, json_extract(json,'\$.update._meta.\"_claude/rateLimit\".unifiedWindows.five_hour.resetsAt') r, json_extract(json,'\$.update._meta.\"_claude/rateLimit\".unifiedWindows.five_hour.utilization') p from transcript where kind = 'update' and json like '%unifiedWindows%' and time >= '<day>T00:00:00Z'),
w as (select r, strftime('%Y-%m-%dT%H:%M:%SZ', r - 18000, 'unixepoch') s, strftime('%Y-%m-%dT%H:%M:%SZ', r, 'unixepoch') e, max(p) top from u where time < strftime('%Y-%m-%dT%H:%M:%SZ', r, 'unixepoch') group by r)
select s, top, (select min(time) from u where u.r = w.r and u.p = w.top) top_at, (select round(sum(cost_usd), 1) from turn_usage where harness = 'claude-code' and ended_at between w.s and w.e) usd from w order by s desc"
```

First prompt of a session:

```sh
sqlite3 -readonly ~/.mobius/mobius.db "select json_extract(json, '$.text') from transcript where session = <id> and kind = 'prompt' order by id limit 1"
```

## Server log

The running Mobius writes its log to stderr and to `<data_dir>/logs/mobius.log`. The default `data_dir` is `~/.mobius`. At 10 MB, Mobius renames the file to `mobius-<UTC time>.log` and starts a new `mobius.log`. It keeps 14 old files and deletes the oldest file. A release before pull request #2102 writes no log file.

The time at the start of each line uses the local time zone of the machine of Mobius, not UTC. On the machine of Mobius, this zone is Central European time (CET or CEST).

The glob `mobius*.log` sorts the old files first and `mobius.log` last, so the lines stay in time order.

Each poll of a repository writes the line `poll <owner>/<repo>: <duration>`. A failed poll writes one more line `poll <owner>/<repo>: <error>`. Show only the poll errors of one repository:

```sh
grep -h 'poll <owner>/<repo>: ' ~/.mobius/logs/mobius*.log | grep -vE ': ([0-9]+m)?[0-9.]+(ms|s)$'
```

Show the last lines of the log:

```sh
tail -n 200 ~/.mobius/logs/mobius.log
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
