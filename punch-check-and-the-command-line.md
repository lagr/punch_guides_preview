# `punch check` and the Command Line

Applies to Punch 0.1.

A Punch app is checked by one command, `punch check`, which runs everything CI runs, the same way on a laptop: formatting, lint, stale generated files, query plans, migrations, the build, the API contract, job payloads, and the specs. Around it is a small command line for the daily loop and for running the app. This guide shows the two binaries, the loop, what each check protects, how to read a failure, and how to drive all of it from scripts and agents.

After reading this guide, you will know:

- which commands belong to the `punch` tool and which to the app's binary;
- the development loop: `punch dev`, `punch typecheck`, and specs;
- what each step of `punch check` protects, and the honest fix when it fails;
- which files are generated, and why they are committed;
- how to run the same checks in CI;
- how scripts and agents use `--json` and exit codes.

Before you start: you have read [How a Punch App Works](how-a-punch-app-works.md). The examples are run in the reference app and in the guides' examples app.

## Two binaries, one entry point

You work with two programs:

- **`punch`**, the standalone tool. It builds, generates, and checks: `punch new`, `punch generate`, `punch dev`, `punch typecheck`, `punch check`, `punch db:migration`, `punch db:generate`, `punch i18n:generate`, `punch openapi`, `punch jobs:lock`.
- **The app's own binary**, such as `bin/invoicing`. It runs and inspects the app: `serve`, `jobs`, `db:migrate`, `db:audit`, `tokens:create`, `roles:assign`, `config`, `ops`, `backup`.

You rarely have to choose. `punch` forwards a command it does not have to the app's binary: `punch tokens:create` builds the app and runs `bin/invoicing tokens:create`. In production only the app's binary exists. `punch help` lists every command with a line on what it does:

```console
$ punch help
Usage: punch <command> [arguments] [--json]

Commands:
  help               List commands
  version            Print the Punch and Crystal versions
  config             Show the settings from the environment (PUNCH_STORAGE, PUNCH_PORT, ...), checked as at boot
  db                 Show the linked SQLite and, given a file, its state: punch db [PATH]
  db:shell           Open the SQLite shell on a database: punch db:shell PATH [SQL]
  db:migration       Create a migration file: punch db:migration NAME, or punch db:migration --punch outbox|idempotency|webhooks|auth|roles [--users TABLE] for one of Punch's own tables
...
```

A typo gets a suggestion, and the exit code 2 of a usage error:

```console
$ punch rotues
punch: Unknown command 'rotues'. Did you mean 'routes' or 'roles'? Run 'punch help' for a list.
```

**The tool matches the app's Punch.** The generators live in the tool, so a `punch` from another Punch version would write files the app does not expect. Build it from the app's `lib/punch` after `shards install`, and put `lib/punch/bin` on the `PATH`:

```console
$ crystal build lib/punch/src/cli/main.cr -o lib/punch/bin/punch --release
```

A `punch` of another version runs the app's own copy instead when there is one, and refuses with `tool_mismatch` when there is not.

## The development loop

Three commands make the loop, from fastest to most complete:

1. **`punch typecheck`** after every edit. It regenerates queries, translations, and the require file, then type-checks without building: a few seconds, with every error and its location. `--json` gives the errors as data.
2. **`crystal spec spec/billing/invoices/`** for the area you work on.
3. **`punch check`** before you are done.

While working on pages, **`punch dev`** keeps the app running. Each round it applies pending migrations, regenerates what changed, rebuilds, and restarts the app, then watches `app/`, `src/`, `db/`, `config/`, and `shard.yml` for the next change. It runs the app with `PUNCH_TRACE=console`, so each request prints its trace as a tree ([How a Punch App Works](how-a-punch-app-works.md#one-request-end-to-end)). A failed step prints its error and leaves the running app as it was; fix and save.

`punch dev` does not regenerate `openapi.json` on every change, because that needs a second build. Run `punch openapi` when the API changed; `punch check` reports a stale one.

## `punch check`

`punch check` runs every step that applies to the project, and runs all of them even after one fails, so one run reports everything. A line per step goes to stderr as it finishes. Here it is in an app with a new namespace whose outline is not written yet:

```console
$ punch check
passed  format        0.1 s  src, spec, app
passed  schema        0.0 s  db/schema.sql matches the migrations.
passed  migrations    0.0 s  6 migrations: 6 instant, 0 scan, 0 rebuild; 0 new since main
passed  punchtables    0.0 s  outbox, idempotency, auth, roles as Punch expects
passed  queries       0.0 s  current   app/accounts/users/records.gen.cr
passed  explain       0.0 s  2 queries, 2 statements: no full scans or sorts
passed  i18n          0.0 s  current   app/translations.gen.cr
failed  requires      0.0 s
passed  unreached     0.6 s  66 files, all required
skipped docs          0.0 s  nothing to check
failed  lint          0.1 s
passed  build        17.5 s  bin/examples (17.5 s)
passed  creds         0.0 s  checked development, test
passed  limits        0.0 s  5 commands without a required user limited or exempt; 8 operations limited
passed  policies      0.0 s  48 operations with a policy, 10 public
passed  events        0.0 s  12 events published, 4 subscriptions
skipped webhooks      0.0 s  nothing to check
passed  jobs          0.0 s  17 job types, 10 recurring
passed  boot          0.0 s  6 migrations embedded, applied at boot
passed  payloads     10.5 s  current   payloads.lock.json (17 job types, 16 payloads); against main: unchanged
passed  openapi       1.3 s  current   openapi.json (14 operations, 24 schemas); against main: No breaking changes.
passed  specs        15.8 s  26 examples, 0 failures, 0 errors, 0 pending (CRYSTAL_WORKERS=4)
passed  nocache       0.2 s  26 examples, 0 failures, 0 errors, 0 pending (PUNCH_CACHE=off CRYSTAL_WORKERS=4)
punch: 2 checks failed: requires, lint

== requires
app/requires.gen.cr does not match the files under app/. Run 'punch requires:generate' and commit the result; never edit it by hand.

== lint
Inspecting 88 files

.................................................F......................................

app/billing/disputes.cr:1:1
[C] Punch/NamespaceFile: `punch generate` left this outline for you to write: say what `Billing::Disputes` means and is for, in plain words, in place of this line (lib/punch/docs/models.md)
> # punch generate: say here what Billing::Disputes means in this app, what it is
  ^

Finished in 57.94 milliseconds
88 inspected, 1 failure
```

Each failed step's output follows the summary, and each message says what to do. The exit code is 1.

## What each step protects

| Step | Protects against | The honest fix |
|---|---|---|
| `format` | Diffs that are only formatting | `crystal tool format` |
| `schema` | `db/schema.sql` out of step with the migrations | `punch db:migrate`; never edit the schema file |
| `migrations` | A migration that rebuilds a large table without saying so | Rethink the migration, or acknowledge the rebuild with `-- rebuild: <table>` ([Migrations](migrations.md)) |
| `punchtables` | Punch's own tables differing from what this Punch expects | Write the migration the message names |
| `queries` | Generated query files older than their SQL | `punch db:generate` |
| `explain` | A query that scans a whole table or sorts without an index | Add an index or change the query; `-- scan:` only with a real reason ([Keeping SQL Safe and Fast](keeping-sql-safe-and-fast.md)) |
| `i18n` | A language missing a text; a code without a message | Add the keys ([Translations and Formats](translations-and-formats.md)) |
| `requires` | A new file that is not part of the build; a file of no known kind | `punch requires:generate`; rename a file of no kind ([Application Structure](application-structure.md)) |
| `unreached` | Code that compiles away because nothing requires it | Require it, usually by generating the require file |
| `lint` | Code that compiles but breaks a convention | Follow the message; disable a rule on one line only with a reason |
| `build` | Compile errors and warnings | Fix them; warnings fail too |
| `creds` | A committed key file; credentials that do not open | Keep `config/credentials/*.key` out of git ([Credentials and Keys](credentials-and-keys.md)) |
| `limits` | A Command without a signed-in user and without a rate limit | `@[Punch::RateLimit]`, or `@[Punch::RateLimitExempt("reason")]` with a real reason |
| `policies` | An operation without a policy | `@[Punch::Authorize(Policy)]` or `@[Punch::Public]` |
| `events` | An event no operation publishes | Publish it or delete it |
| `webhooks` | A webhook provider that does not verify, added unnoticed | Never fails; read the list |
| `jobs` | A recurring job whose schedule does not parse | Fix the schedule |
| `boot` | A binary without its migrations, which would not migrate at boot | `Punch.migrations` in `app/setup.cr` |
| `payloads` | A change to a job's payload that stored jobs cannot be read with | Make the field optional or defaulted, or raise the payload version ([Background Jobs](background-jobs.md)) |
| `openapi` | A stale `openapi.json`; a breaking API change against the main branch | `punch openapi`; for a break, rethink it or raise the version ([Building a JSON API](building-a-json-api.md#breaking-changes)) |
| `specs` | Behaviour | Fix the code, or the spec if the behaviour changed on purpose |
| `nocache` | Code that is only correct with a warm cache | Make it correct with the cache off ([Caching](caching.md)) |

Several steps compare against the main branch, so a change is judged against what is deployed: `migrations`, `payloads`, and `openapi`. `--base REF` names another base; by default it is `origin/main`, else `main`.

> [!IMPORTANT]
> Never make a check pass by weakening it: no `-- scan:` without a real reason, no blanket `ameba:disable`, no edited generated file, no rate-limit exemption without a reason a reviewer would accept. The checks are there because each one guards against a failure that has happened in real apps.

## Generated files are committed

Several files are written by tools from sources you edit. They are committed, so a reviewer sees what a change does to the schema, the API, or the job payloads, and each is checked for staleness:

| File | Written by |
|---|---|
| `db/schema.sql` | `punch db:migrate` |
| `queries.gen.cr`, `records.gen.cr` | `punch db:generate` |
| `app/translations.gen.cr` | `punch i18n:generate` |
| `app/requires.gen.cr` | `punch requires:generate` |
| `openapi.json` | `punch openapi` |
| `payloads.lock.json` | `punch jobs:lock` |

Never edit one by hand. Change its source and run its generator; `punch typecheck` and `punch dev` run the first four for you.

## Inspecting what Punch derived

Much of an app is derived from names and types. Three commands show what Punch made of it:

- **`punch routes`** lists every route with the surfaces it answers, the operation, the policy, the page holding a Command's form, rate limits, and entry points for agents:

  ```console
  $ punch routes
  ...
  GET     /billing/invoices/:invoice_id         json                Billing::Invoices::Show  (policy: Billing::Invoices::ByRole)
  GET     /billing/invoices/:invoice_id         html,markdown,live  Billing::Invoices::ShowPage  (policy: Billing::Invoices::ByRole)
  POST    /billing/invoices/:invoice_id/send    json,html,markdown  Billing::Invoices::Send  (form on Billing::Invoices::ShowPage)  (policy: Billing::Invoices::ByRole)  (limited: billing.invoices.send.actor)
  ...
  ```

- **`punch events`** lists each event with who publishes it and who subscribes.
- **`punch authz:explain OPERATION --user ID`** says why a user may or may not run an operation.

`punch config` shows the app's settings as the app would read them at boot, with every problem at once ([Configuring Punch Applications](configuring-punch-applications.md)).

## For scripts and agents

Every command follows the same conventions, so a script or an agent can drive it without parsing text:

- **`--json`** puts exactly one JSON document on stdout, also on failure.
- **Exit codes:** 0 for success, 1 for failure, 2 for a usage error.
- **Failures** are `{"error": {"code", "message", "details"}}`:

  ```console
  $ punch rotues --json
  {"error":{"code":"unknown_command","message":"Unknown command 'rotues'. Did you mean 'routes' or 'roles'? Run 'punch help' for a list.","details":{"did_you_mean":["routes","roles"]}}}
  ```

`punch check --json` gives every step with `name`, `status` (`passed`, `failed`, `skipped`), `seconds`, and `detail`; an agent finishes its work when every step is `passed` or `skipped`. `punch routes --json` gives each route as data:

```json illustrative="one route of the reference app's punch routes --json"
{"method": "POST", "path": "/billing/invoices/:invoice_id/send", "operation": "Billing::Invoices::Send", "operation_id": "billing.invoices.send", "kind": "command", "surfaces": ["json", "html", "markdown"], "form_page": "Billing::Invoices::ShowPage", "actor": "required", "rate_limits": [{"name": "billing.invoices.send.actor", "limit": 30, "period_seconds": 3600.0, "by": "actor", "store": "memory"}], "policy": "Billing::Invoices::ByRole", "public": false, "unlisted": false, "entry": false}
```

The app's `AGENTS.md`, which `punch new` writes, tells an agent the same loop: `punch typecheck` after every edit, specs for the area, `punch check --json` before finishing, and never skipping a check to make it pass.

## Running the checks in CI

CI runs the same command. `punch new` writes a GitHub Actions workflow that installs the Crystal version from `.tool-versions`, installs shards, builds the tool from `lib/punch`, and runs `punch check`:

```yaml illustrative="the steps of .github/workflows/check.yml, as punch new writes it"
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
- run: shards install
- name: Build the punch tool that matches lib/punch
  run: crystal build lib/punch/src/cli/main.cr -o lib/punch/bin/punch --release
- run: lib/punch/bin/punch check
  env:
    CRYSTAL_WORKERS: "4"
```

The checks that compare against the main branch need its history, so the workflow checks out with `fetch-depth: 0`. Without a base revision, `openapi` and `payloads` pass with "no base revision to compare with", and `migrations` treats every migration as new. The HTML matchers of `punch/spec` need `libxml2`, which the workflow installs.

## Common mistakes

**Running a `punch` from another Punch version.** It refuses with `tool_mismatch`, or runs the app's own copy. Build the tool from `lib/punch`.

**Fixing a check by editing what it checks.** Editing `openapi.json` or `db/schema.sql` by hand, or a generated `.gen.cr` file, makes the next generator run undo it, and the check fail again. Change the source.

**Skipping `punch check` because the specs pass.** Specs do not see a stale `openapi.json`, a query that scans a table, a migration that rebuilds one, or a breaking payload change. Run the whole check.

**Silencing lint for a whole file.** A rule disabled at the top of a file hides every later violation there. Disable one line, with a comment that says why.

**Reading text output in a script.** Text output may change wording. Use `--json`.

## Limits

- `punch check` checks one app; it does not check across services.
- The lint rules do not cover every convention yet; the app's `AGENTS.md` lists the rules you keep by hand.
- `punch dev` rebuilds the whole app on each change; there is no hot reload.

## Rules

- Run `punch typecheck` after every edit and `punch check` before you are done. ([The development loop](#the-development-loop))
- Fix what a check reports; never weaken the check. ([What each step protects](#what-each-step-protects))
- Commit generated files with the change that caused them, and never edit them by hand. ([Generated files are committed](#generated-files-are-committed))
- Use the `punch` built from the app's `lib/punch`. ([Two binaries, one entry point](#two-binaries-one-entry-point))
- Drive commands from scripts and agents with `--json` and exit codes. ([For scripts and agents](#for-scripts-and-agents))
- Run the same `punch check` in CI as on your machine. ([Running the checks in CI](#running-the-checks-in-ci))

## Reference

- [Tooling](../tooling.md): every command of the tool and the app, and every setting.
- [`punch check` steps](../reference/check-steps.md): each step and when it runs.
- [Lint rules](../reference/lint-rules.md): Punch's Ameba rules.
- [Compile errors](../reference/compile-errors.md): every message of Punch's compile-time checks.
