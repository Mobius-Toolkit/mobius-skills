---
name: go-test-fails-without-fix
description: Show that a new Go test of Mobius-Toolkit/Mobius fails without the fix and passes with it, with no `git stash`. Use it after you commit the fix and the test, as the rule "A new test must fail on the base commit" of `AGENTS.md` asks.
metadata:
  roles: implementer
---

# A new Go test fails without the fix

`AGENTS.md` tells that a new test must fail on the base commit and pass on the commit of the change. This procedure removes the fix from the files for one test run and then puts it back from the commit. The test files stay.

WARNING: Do not use `git stash` for this check. All worktrees of the bare clone share one stash stack. `git stash pop` can apply and drop the stash of a different session.

1. Commit the fix and the new tests first. Run `make fmt` before the commit, as `AGENTS.md` tells.

2. Set the values:
   - `<files>`: the source files of the fix, with no `_test.go` file. Do not give a test helper that the new test needs.
   - `<package>` and `<names>`: the package and the names of the new tests.

3. Remove the fix, run the new tests, and put the fix back, in one Bash call:

   ```sh
   b=$(git merge-base HEAD origin/main)
   git diff $b HEAD -- <files> | git apply -R && go test ./<package> -run '<names>' 2>&1 | tail -15
   git checkout HEAD -- <files> && git status --short
   ```

   - The new tests must fail in the output. A compile error of a test, because a new function of the fix is not there, is also a failure on the base commit.
   - `git status --short` must show no file of `<files>`. Else run `git checkout HEAD -- <files>` again.

4. Run the new tests one time on the commit. They must pass.

## Limits

- Run only the new tests with `-run`. Do not run the full `internal/engine` package.
- Give a Go test of `internal/engine` the Bash `timeout` 600000.
- Run each of the two checks one time, as `AGENTS.md` tells.
