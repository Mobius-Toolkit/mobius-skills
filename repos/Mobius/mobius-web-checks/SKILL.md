---
name: mobius-web-checks
description: Check a change of the web app of Mobius-Toolkit/Mobius. Use it to run one Playwright e2e test in the foreground and read why it failed. Also use it to run a pure TypeScript module of `web/src/lib` with Node, with no test runner.
metadata:
  roles: implementer
---

# Checks of the web app of Mobius

The web app is in the folder `web` of Mobius-Toolkit/Mobius. Run the commands below from `web`.

## Run one e2e test

The script `pnpm e2e` is `pnpm build && playwright test`. The arguments go to `playwright test`.

The projects in `playwright.config.ts` have dependencies. A project runs only after its dependency projects, because it needs their data. Thus one spec file also runs the full chain of its dependency projects. A run of `e2e/actions.spec.ts` takes about 5 minutes.

Do not use `--no-deps`. A project that runs without its dependency projects does not have the data that it needs.

Do these steps:

1. Give the Bash call the maximum timeout of 600000 ms. A command that runs longer than the timeout of the Bash call goes to the background.
2. Keep the shell `timeout` below 600 seconds, so that the run stops before the Bash call stops.
3. Write the output to a new temporary file, and then show the end of the file. Other worktrees run e2e tests at the same time, so do not use a fixed file name.

   ```sh
   o=$(mktemp); (timeout 590 pnpm e2e e2e/<spec>.spec.ts -g "<part of the test name>" > $o 2>&1; tail -30 $o)
   ```

4. When a test fails, read its `error-context.md`. It has the error and the page snapshot at the time of the failure:

   ```sh
   ls test-results
   sed -n 1,80p test-results/<folder of the test>/error-context.md
   ```

Do not run `pnpm e2e` in the background and then poll the output with `sleep`. The foreground call above gives the result in one step.

The `-g` filter is only for the read of one failure. Before a push, run the full check of `AGENTS.md`: `pnpm e2e e2e/<changed>.spec.ts e2e/screenshots.spec.ts`, and read the screenshots of each changed screen.

## Run a pure module of web/src/lib with Node

Node 24 runs a `.ts` file with no flag, because it removes the types. Node does not know the `@/` alias of `tsconfig.json`, and an import must have the `.ts` extension. Thus copy the modules to a temporary folder, and change each `@/lib/<name>` import to `./<name>.ts`.

Use this procedure only for a module with no import of React or of a browser API.

```sh
d=$(mktemp -d)
sed 's#"@/lib/\([a-zA-Z]*\)"#"./\1.ts"#' src/lib/<module>.ts > $d/<module>.ts
cp src/lib/<imported module>.ts $d/
cat > $d/t.ts <<'EOF'
import { <function> } from "./<module>.ts";
console.log(JSON.stringify(<function>(<input>)));
EOF
node $d/t.ts
rm -rf $d
```

Copy only a module of a value import. Node removes an `import type` line, so its module needs no copy. Do the `sed` change also on each copied module that has an `@/lib/` import. Do not put the probe script in the worktree, so that it does not go into a commit.
