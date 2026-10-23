# The Ops Console and `punch ops`

Applies to Punch 0.1.

A Punch app records what its operators need in its own storage directory: errors grouped by cause, every job run, performance per route, job, and named query, the outbox, the backup's state. No third-party service is involved. The ops console at `/_punch` shows it to people, as markdown to agents, and as JSON to programs, and `punch ops` shows the same on the command line, on the box or against a running app.

After reading this guide, you will know:

- who may open the console, and how to give the first operator access;
- how to find and fix a production problem with it;
- how to be told when something goes wrong;
- how to let an agent watch and triage.

Before you start: you know what a wide event is ([Tracing and Metrics](tracing-and-metrics.md)) and how jobs fail and are retried ([Background Jobs](background-jobs.md)).

## Who may open it

The console is a set of Punch's own pages and Commands, mounted once in `app/setup.cr`. Its policy asks the roles: whoever holds a global role that permits the console's operations may use it. `punch new` writes that role, `Accounts::Roles::Operator`:

```crystal source=docs/guides/examples/app/accounts/roles.cr
# app/accounts/roles.cr
module Accounts::Roles
  # Runs the app: the ops console at /_punch. Given on the box with `punch roles:assign
  # Accounts::Roles::Operator --email ADDRESS`; `db/seeds.cr` gives it to
  # the development user.
  struct Operator
    include Punch::Role

    permits Punch::Ops::Console::Operations
  end
end
```

Nobody holds it at first. Give the first operator on the box:

```console
$ bin/examples roles:assign Accounts::Roles::Operator --email ada@example.org
```

`db:seed` gives it to the development user, so the console is open in `punch dev` right away. For anyone else, the console answers 404: it does not show that it exists, and links to it are left out of their pages.

## What it shows

| Page | Shows | Actions |
|---|---|---|
| `/_punch` | open errors, failed job runs, jobs per state, worker processes, requests in the last hour; a banner while a restore waits | |
| `/_punch/errors` | error groups, most recently seen first; each with its backtrace and the wide events of its latest occurrences | resolve, reopen |
| `/_punch/jobs` | jobs per queue and state, failed jobs by type with the latest error, recurring jobs with their next run | retry, discard, pause and resume a queue |
| `/_punch/performance` | per route, job type, or named query: runs, failures, percentiles, total time; the slowest requests | |
| `/_punch/events` | the outbox, and each event's publishers, subscribers, and their failures | |
| `/_punch/database` | each file's size and WAL, writes waiting, checkpoints, the replica's lag, the last restore check | |
| `/_punch/recovery` | the reconciliation after a restore | confirm |
| `/_punch/keys` | the key ring's keys and their use, where each credential came from | |
| `/_punch/webhooks` | providers and their deliveries, each stored request | retry, replay |
| `/_punch/mail` | mail captured in development | |
| `/_punch/cache` | each cache store's hits, misses, and size | clear |

**Errors are grouped by cause**: the exception's class, where it was handled, and the files of the top of its backtrace. A deploy that moves code keeps the group, and different messages from the same bug do not split it. A resolved group opens again when the error comes back.

Everything is kept for 14 days by default (`PUNCH_OPS_RETENTION_DAYS`), open error groups for as long as they are open.

## On the command line

`punch ops` shows the overview, on the box or in a checkout:

```console
$ bin/examples ops
errors      0 open, 0 occurrences in 24 h
job runs    0 of 2 failed in 24 h
jobs        0 ready, 0 scheduled, 0 running, 0 blocked, 0 failed, 0 incompatible
processes   0
requests    2 in the last hour, 0 failed, average 10.93 ms
```

The other commands follow the pages: `ops:errors` and `ops:error ID`, `ops:resolve ID`, `ops:jobs` and `ops:job ID`, `ops:retry ID`, `ops:discard ID`, `ops:performance`, `ops:slow`, and `ops:record ID` for one stored wide event:

```console
$ bin/examples ops:performance --window day
Requests, last 24 hours, the most time spent first
    runs failed        p50        p95        p99        max      total  name
       1      0   20.37 ms   20.37 ms   20.37 ms   20.37 ms     0.02 s  GET /accounts/users/me
       1      0    1.49 ms    1.49 ms    1.49 ms    1.49 ms      0.0 s  GET /accounts/sessions/new
```

All take `--json`. The `punch` tool also reads a running app from elsewhere: `punch ops --url https://invoices.example.org`, with an operator's API token in `PUNCH_OPS_TOKEN`.

## Three runbooks

**A new error group.**

1. Open it in `/_punch/errors`, or `punch ops:error ID`: the exception, where it was handled, the backtrace.
2. Open the latest occurrence's wide event: the route or job, the outcome, the input's errors, the queries it ran.
3. Fix the code, add a spec that reproduces it, and deploy. Then resolve the group; it opens again if the fix did not hold.

**Jobs that keep failing.**

1. `/_punch/jobs` groups failed jobs by type with their latest error; open one for its payload and runs.
2. If the cause was outside the app, such as a provider that was down, retry all failed jobs of the type.
3. If the payload is wrong, fix the code, deploy, then retry; or discard what should not run.
4. Jobs parked as incompatible need a binary that reads their payload version ([Background Jobs: changing a payload](background-jobs.md#changing-a-payload)).

**A backup that is behind.**

1. `/_punch/database`, or `bin/<app> backup`: which file, how far behind, whether Litestream answers.
2. Litestream's own log lines are JSON on standard error, beside the app's.
3. Fix the storage provider's access or the network; the sidecar restarts Litestream by itself ([Backup and Restore](backup-and-restore.md#is-it-working)).

## Being told

Trouble is announced as domain events, so alerting is a subscriber of the app's, not a separate system:

| Event | Published when |
|---|---|
| `Punch::Ops::ErrorOccurred` | an error group opens, or one that was resolved recurs |
| `Punch::Ops::JobsFailing` | jobs of a type have failed for good or are parked |
| `Punch::Ops::BackupStale` | a file is more than five minutes behind, or the restore check has not passed for 26 hours |

Each trouble is announced once, until it has recovered. A deferred subscriber mails the operators or calls a pager:

```crystal illustrative="an app's alerting subscriber; the reference app has none"
@[Punch::Subscriber::Delivery(:deferred)]
struct Ops::Page
  include Punch::Subscriber(Punch::Ops::ErrorOccurred)
  include Punch::Subscriber(Punch::Ops::JobsFailing)
  include Punch::Subscriber(Punch::Ops::BackupStale)

  def call(event, job : Punch::Job::Context) : Punch::Job::Result
    Alerts.page(event, job)
  end
end
```

## For agents

Every console page is readable by agents. An agent with an operator's API token reads `/_punch/errors.md`, or any console path with `Accept: text/markdown`, and gets the page as markdown, with links to each error's page and the buttons as action blocks it can use. The same paths answer JSON to programs.

That makes a loop an agent can run: read an open error, follow its stored wide event to the route or job and the named query, propose a fix with a spec that reproduces it, and let `punch check` verify it. Give the agent its own user and its own token, so its requests carry its own user ID on their wide events, and revoke the token when it is done ([Authentication](authentication.md#sessions-and-api-tokens)).

## Common mistakes

**Giving everyone the operator role.** The console shows stored requests and credentials' sources. Give it to the people who run the app, on the box, with `roles:assign`.

**Resolving an error without a fix.** It reopens at the next occurrence, and the alert fires again. Resolve after the deploy that fixes it.

**Retrying all failed jobs before reading one.** If the payload is the problem, they all fail again. Open one first.

**Alerting from an after-commit subscriber.** Paging someone is a call to another system. Make the subscriber deferred.

## Limits

- The console reads the process that serves it: cache counters and metrics are that process's; errors, job runs, and performance cover every process writing to the same `ops.db`.
- Retention is by age; there is no export of `ops.db` beyond the JSON reports.

## Rules

- Give the operator role only to the people who run the app, on the box with `roles:assign`. ([Who may open it](#who-may-open-it))
- Find a problem by its error group and its stored wide event; resolve a group after the fix is deployed. ([Three runbooks](#three-runbooks))
- Alert with a deferred subscriber to `Punch::Ops::ErrorOccurred`, `JobsFailing`, and `BackupStale`. ([Being told](#being-told))
- Give an agent its own user and API token. ([For agents](#for-agents))

## Reference

- [Ops data](../ops.md): what is recorded, retention, every page and its actions, the JSON reports, notifications, and every command.
- [Authorization](../authorization.md): the global role that opens the console.
