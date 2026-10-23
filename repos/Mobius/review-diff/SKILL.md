---
name: review-diff
description: Read the diff of a task in Mobius-Toolkit/Mobius without the generated files. Use it at the start of a review, or when you must see what a branch or a pull request changed.
---

# Diff of a task without the generated files

In Mobius-Toolkit/Mobius, `make generate` writes these files. Do not review them line by line.

| Generated file | Source |
| --- | --- |
| `internal/store/db.go`, `internal/store/models.go`, `internal/store/*.sql.go` | `internal/store/queries.sql` and `internal/store/migrations/` (sqlc) |
| `api/openapi.json` | the handlers in `internal/api` |
| `web/src/api/api.gen.ts` | `api/openapi.json` (orval) |

`.mobius/check` runs `make generate` and fails when a generated file is not current. Thus a head that passed the check has current generated files.

Set `<base>` and `<head>` to the two commits of the review.

1. Read the list of the changed files, and the full messages of the commits. The body of the last commit can hold a required report.

   ```sh
   git diff --stat <base> <head>
   git log --format='%H%n%B----' <base>..<head>
   ```

2. Read the diff of the source files with no generated files and no tests.

   ```sh
   git diff <base> <head> -- . ':!internal/store/db.go' ':!internal/store/models.go' ':!internal/store/*.sql.go' ':!api/openapi.json' ':!web/src/api/api.gen.ts' ':!*_test.go' ':!web/e2e'
   ```

   When the output is too long, give the directories one at a time, for example `-- internal/engine ':!*_test.go'`.

3. Read the diff of the tests.

   ```sh
   git diff <base> <head> -- '*_test.go' web/e2e
   ```
