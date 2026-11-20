---
name: mobius-web-checks
description: Run `bun run e2e` for one spec or one test of the web app of Mobius-Toolkit/Mobius in the foreground, with the `MOBIUS_E2E_GROUP` of the spec, and read why a test failed. Also run a pure TypeScript module of `web/src/lib` with `bun -e`, with no test runner. Use it before each `bun run e2e` or `playwright test` run, and before a check of a function of `web/src/lib`.
metadata:
  roles: implementer
---

# Checks of the web app of Mobius

The web app is in the folder `web` of Mobius-Toolkit/Mobius. Run the commands below from `web`. Use Bun. Do not use npm, yarn, pnpm or Node.

## Run one e2e test

The script `bun run e2e` builds the UI and then runs `bunx --bun playwright test`. The arguments go to `playwright test`.

The projects in `playwright.config.ts` form four groups: `screenshots`, `a`, `b` and `c`. Each group has its own server. With no `MOBIUS_E2E_GROUP`, the run starts all four servers. Set `MOBIUS_E2E_GROUP` to the group of the spec, so that only its server starts:

```sh
grep -n '"<spec>.spec.ts"' playwright.config.ts
```

The line starts with `groupA`, `groupB` or `groupC`. The spec `screenshots.spec.ts` is in the group `screenshots`.

The projects have dependencies. A project runs only after its dependency projects, because it needs their data. Thus one spec file also runs the full chain of its dependency projects. A run of `e2e/actions.spec.ts` takes about 5 minutes.

Do not use `--no-deps`. A project that runs without its dependency projects does not have the data that it needs.

Do these steps:

1. Give the Bash call the maximum timeout of 600000 ms. A command that runs longer than the timeout of the Bash call goes to the background.
2. Keep the shell `timeout` below 600 seconds, so that the run stops before the Bash call stops.
3. Write the output to a new temporary file, and then show the end of the file. Other worktrees run e2e tests at the same time, so do not use a fixed file name.

   ```sh
   o=$(mktemp); (MOBIUS_E2E_GROUP=<group> timeout 590 bun run e2e e2e/<spec>.spec.ts -g "<part of the test name>" > $o 2>&1; tail -30 $o)
   ```

4. When a test fails, read its `error-context.md`. It has the error and the page snapshot at the time of the failure:

   ```sh
   ls test-results
   sed -n 1,80p test-results/<folder of the test>/error-context.md
   ```

Do not run `bun run e2e` in the background and then poll the output with `sleep`. The foreground call above gives the result in one step.

The `-g` filter is only for the read of one failure. Before a push, run the full check of `AGENTS.md`: `bun run e2e e2e/<changed>.spec.ts e2e/screenshots.spec.ts`, and read the screenshots of each changed screen. Two specs of different groups need two runs, one for each `MOBIUS_E2E_GROUP`.

## Run a pure module of web/src/lib with Bun

Bun runs TypeScript and reads the `@/` alias of `web/tsconfig.json`. Thus `bun -e` in `web` imports a module of `src/lib` with no copy and no file in the worktree:

```sh
bun -e 'import { <function> } from "./src/lib/<module>.ts"; console.log(JSON.stringify(<function>(<input>)));'
```

Use this procedure only for a module with no import of React or of a browser API.

