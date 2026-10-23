# Background Jobs

Applies to Punch 0.1.

Some work does not belong in a request: mailing a customer, calling a payment provider, sweeping a table every morning. It is slow, it can fail for reasons outside the app, and the person who clicked should not wait for it. Punch runs such work as jobs: typed structs stored in `queue.db`, a SQLite file beside the app's database, run by workers in the app's own process and retried when they fail. There is no Redis and no separate service to run.

After reading this guide, you will know:

- how to move slow or failure-prone work out of a request into a job;
- how to make a job safe to run more than once;
- how to enqueue from a Command, so the job runs exactly when the Command's writes commit;
- how to run work on a schedule, and limit how many jobs run at once;
- how to change a job's payload without breaking the jobs already waiting;
- how to inspect, retry, and discard jobs, and how to test them.

Before you start: you know what a Command is ([Operations and routing](../operations.md)) and how inputs and domain types work ([Input and Domain Types](input-and-domain-types.md)). The examples come from the reference app (`reference/`) and, for the features it does not use, from the guides' examples app (`docs/guides/examples/`).

## What a job is

A job is a struct that includes `Punch::Job`. Its fields are the payload: Punch stores them as JSON when the job is enqueued, and parses them back into the struct when a worker runs it. `perform` does the work.

The reference app records a reminder when a sent invoice comes due unpaid:

```crystal source=reference/app/billing/invoices/jobs/remind.cr
# app/billing/invoices/jobs/remind.cr
module Billing::Invoices
  # Comes due with its invoice: `Send` enqueues it through the outbox for
  # the due date (lib/punch/docs/jobs.md). It records a reminder for an invoice that
  # is still unpaid; it mails nothing (the invoice itself is mailed by
  # `MailInvoice`). Safe to run twice: an invoice has one
  # reminder.
  @[Punch::Job::Options(queue: "reminders")]
  struct Remind
    include Punch::Job

    getter invoice_id : ID

    def initialize(@invoice_id)
    end

    def perform(job : Punch::Job::Context) : Punch::Job::Result
      job.write do |db|
        case invoice = Store.load(db, invoice_id)
        in Nil            then discard("invoice #{invoice_id} was deleted")
        in Invoice::Draft then discard("invoice #{invoice_id} is not sent")
        in Invoice::Paid  then discard("invoice #{invoice_id} is paid")
        in Invoice::Sent
          Queries.add_reminder(db, invoice_id: invoice.id, created_at: job.now)
          nil
        end
      end
    end
  end
end
```

`perform` returns a `Punch::Job::Result`, which says how the run ended:

| `perform` returns | Meaning | The job becomes |
|---|---|---|
| `nil` | Done | `done` |
| `retry(reason)`, or `retry(reason, after: 1.hour)` | An expected failure that may pass, such as a provider answering 503 | `ready` again after a backoff; `failed` after the last attempt |
| `discard(reason)` | Nothing left to do: the invoice was deleted or paid | `discarded`, never retried |

`retry` and `discard` make a `Punch::Job::Retry` and a `Punch::Job::Discard`; every job and deferred subscriber has them. An exception is a bug, but it may be a passing one, so Punch logs it with its backtrace, stores it as the job's error, and retries the job as if it had returned `retry`.

A job is not an operation. It may call other systems, which can take seconds, so it holds no transaction while it runs. It opens short ones when it needs the database: `job.read { |db| … }` and `job.write { |db| … }` on `primary.db`.

Keep the payload to IDs and small values, and read the rest when the job runs. A job that runs an hour later should see the invoice as it is then, not as it was when the job was enqueued. A field that holds a whole record does not compile, and neither does a `Punch::Secret`.

> [!NOTE]
> **Coming from Rails:** a job is like an Active Job class, with the arguments as typed fields instead of a `perform(*args)` list, and the queue in SQLite instead of Redis. There is no `perform_later` on a model: a Command enqueues through its outbox (below), so a job never runs for writes that were rolled back.

## Jobs run at least once

A job can run twice. A process can crash after the work and before recording the job as done; a deploy can outlast the shutdown timeout; a backup restore can bring back jobs that had already run. Punch promises that every job runs, not that it runs only once.

So make every job safe to repeat. Check the state first, and do nothing when the work is already done. The examples app's reminder mails the customer of an overdue invoice:

```crystal source=docs/guides/examples/app/billing/invoices/jobs/remind.cr select=Billing::Invoices::Remind elide=V1
# app/billing/invoices/jobs/remind.cr
module Billing::Invoices
  # Mails the customer of one overdue invoice a reminder, and records
  # when. Safe to run again: an invoice that was paid, or reminded in the
  # meantime, is skipped. At most two run at a time, so the reminders of
  # a big sweep leave the mail relay's other connections to sign-in mail.
  @[Punch::Job::Options(queue: "mail", attempts: 5)]
  @[Punch::Job::Concurrency(2)]
  @[Punch::Payloads::Version(2, from: {1 => V1})]
  struct Remind
    include Punch::Job

    getter invoice_id : ID
    getter reminder : Reminder

    def initialize(@invoice_id, @reminder)
    end

    def perform(job : Punch::Job::Context) : Punch::Job::Result
      # On the run's wide event, so a reminder can be found by its invoice.
      Punch::Trace.event["invoice.id"] = invoice_id.value
      Punch::Trace.event["invoice.reminder"] = reminder.to_s.underscore
      invoice = job.read { |db| Table.find(db, invoice_id) }
      return discard("invoice #{invoice_id} is gone") unless invoice
      return discard("invoice #{invoice_id} is paid") if invoice.paid_at
      if reminded = invoice.reminded_at
        return discard("invoice #{invoice_id} was reminded already") if reminded > job.now - REMINDER_INTERVAL
      end

      result = Punch::Mailer.deliver(ReminderMail.new(invoice, reminder), locale: Punch::I18n.locale(nil), job: job)
      job.write { |db| Queries.mark_reminded(db, id: invoice_id, now: job.now) } if result.nil?
      result
    end

    # Version 1: before there was a final reminder, every reminder was a
    # first one. Never change it.
    struct V1
      # ...
    end
  end
end
```

It skips an invoice that is gone, paid, or reminded in the past week, so a second run of the same job mails nobody. It records the reminder only after the mail went out, so a failed delivery is retried and not lost. What remains is the short window between sending and recording: a crash there mails twice. When a provider takes an idempotency key, pass `job.key`, which stays the same across every attempt of the job, and the provider closes that window too.

## Enqueuing from a Command: the outbox

A Command that enqueues a job asks for a handle in `call`, whose type lists every job it may enqueue. The reference app's `Send` schedules the reminder for the day the invoice is due:

```crystal source=reference/app/billing/invoices/operations/send.cr select=Billing::Invoices::Send::Output,Billing::Invoices::Send#call
# app/billing/invoices/operations/send.cr
module Billing::Invoices
  # ...

  # Sends a draft: it gets the next invoice number and is due after the
  # payment term. Only a draft with lines can be sent. Each user sends at
  # most 30 an hour, since each one mails a customer (`MailInvoice`).
  # A client must name each attempt with an idempotency key, so a retry
  # after a lost answer gets that answer instead of `already_sent`.
  @[Punch::Authorize(ByRole)]
  @[Punch::RateLimit(30, per: 1.hour, by: :actor)]
  @[Punch::Idempotency(:required)]
  struct Send < Punch::Command
    # ...

    alias Output = Detail | NotFound | AlreadySent | NoLines

    def call(input : Input, db : Punch::Writer, _actor : Invoicing::Actor, _grant : Punch::Grant(Send),
             events : Punch::Events(InvoiceSent), jobs : Punch::Jobs(Remind), clock : Punch::Clock) : Output
      case invoice = Store.load(db, input.invoice_id)
      in Nil                          then NotFound.new
      in Invoice::Sent, Invoice::Paid then AlreadySent.new(invoice)
      in Invoice::Draft
        now = clock.now

        case sent = invoice.send(Store.next_number(db), now)
        in Invoice::NoLines then NoLines.new
        in Invoice::Sent
          Store.save(db, sent)
          events.publish(InvoiceSent.new(sent.id, sent.customer_id))
          # Through the outbox: the reminder exists exactly when the sent invoice does.
          jobs.enqueue(Remind.new(sent.id), at: sent.due_at)
          Detail.new(sent, now)
        end
      end
    end
  end
end
```

`jobs.enqueue` does not write to `queue.db`. It writes the job to the outbox, a table in `primary.db`, inside the Command's transaction. If the Command fails or raises, the job is rolled back with everything else. If the Command commits, the job will run: right after the commit, the process hands the entry over to `queue.db`. So the reminder exists exactly when the sent invoice does.

The outbox table comes from Punch, like its other tables. `punch new` writes its migration; an older app adds it with `punch db:migration --punch outbox`.

A Query or a Page that asks for `Punch::Jobs` does not compile: only a Command enqueues ([Background jobs: from a Command](../jobs.md#from-a-command-the-outbox)).

## Enqueuing outside a transaction

Code that runs outside any transaction, such as a job, a CLI command, or a script, enqueues directly into `queue.db`:

```crystal illustrative="the three forms of enqueue; the reference app enqueues from Commands only"
Punch::Job::Queue.enqueue(Remind.new(invoice.id))                    # due now
Punch::Job::Queue.enqueue(Remind.new(invoice.id), in: 3.days)        # delayed
Punch::Job::Queue.enqueue(Remind.new(invoice.id), at: invoice.due_at)
```

A running job uses `job.enqueue` for its follow-ups, after the write they follow from. Called inside a write transaction, either one raises, because `queue.db` is a different file and cannot take part in the transaction: the data could roll back while the job stayed, or the job could be lost while the data committed.

## Queues, attempts, and priority

`@[Punch::Job::Options]` sets three things:

| Option | Default | |
|---|---|---|
| `queue` | `default` | A name such as `mail`. A worker process can be limited to some queues, and a queue can be paused |
| `attempts` | `10` | Runs in all, from 1 to 100. Retries wait 15 seconds, then three times longer each time, at most 6 hours: ten attempts span about 17 hours |
| `priority` | `0` | From -100 to 100. Among jobs that are due, higher runs first |

Choose `attempts` from how long the work stays worth doing. A reminder that goes out two days late is still useful; a sign-in link that arrives after an hour is not.

## Running on a schedule

A job with `@[Punch::Job::Recurring]` is enqueued by Punch itself, on a schedule. The examples app sweeps for overdue invoices every morning:

```crystal source=docs/guides/examples/app/billing/invoices/jobs/sweep_overdue.cr
# app/billing/invoices/jobs/sweep_overdue.cr
module Billing::Invoices
  # Every morning, finds the invoices that are overdue and whose customer
  # was not reminded in the last week, and enqueues a reminder for each.
  # It reminds nobody itself: each reminder is a job of its own, retried
  # on its own.
  @[Punch::Job::Recurring(cron: "0 6 * * *")]
  struct SweepOverdue
    include Punch::Job

    def initialize
    end

    def perform(job : Punch::Job::Context) : Punch::Job::Result
      overdue = job.read { |db| Queries.overdue(db, now: job.now, since: job.now - REMINDER_INTERVAL) }
      overdue.each do |invoice|
        job.enqueue(Remind.new(invoice.id, invoice.reminded_at ? Reminder::Final : Reminder::First))
      end
      nil
    end
  end
end
```

- `cron:` takes the usual five fields, `minute hour day-of-month month day-of-week`, with ranges, steps, lists, and names such as `MON-FRI`. `every: 15.minutes` runs at multiples of the interval.
- Schedules are in UTC.
- Every process with workers runs the schedules, and each run is still enqueued once: its key is the job type and the scheduled time.
- Runs missed while no process was up are not made up. The next scheduled run happens as usual.
- `punch check` parses every schedule (its `jobs` step), so a typo fails before a deploy.

A recurring job is enqueued with an empty payload, so its fields need defaults. The sweep has none, and still defines an empty `initialize`, because a job's payload parser replaces the default one:

```crystal fails=spec/fail/job_without_initialize.cr
struct Export
  include Punch::Job

  def perform(job : Punch::Job::Context) : Punch::Job::Result
  end
end
```

```text
Error: Export is a job, so it defines `initialize`, also when it has no fields (`def initialize; end`): JSON::Serializable, which reads its payload, replaces the default one (lib/punch/docs/jobs.md).
```

The sweep does not mail anyone itself. It enqueues one `Remind` per invoice, so each reminder succeeds, fails, and is retried on its own, and a sweep of five hundred invoices is five hundred small jobs rather than one long one.

Punch brings recurring jobs of its own, such as purging finished jobs and expired sessions every hour and checking the backup every night. `punch jobs:types` lists them with the app's, each with its next run.

## Limiting how many run at once

`@[Punch::Job::Concurrency]` caps how many jobs of a kind run at the same time, across every process. The examples app's `Remind` above takes `@[Punch::Job::Concurrency(2)]`: after a big sweep, at most two reminders talk to the mail relay at once, and sign-in mail still gets through.

The limit has a key. By default it is the job's type. Two options change it:

```crystal illustrative="the two other forms of the key; the examples app uses the plain limit"
@[Punch::Job::Concurrency(1, by: :invoice_id)]   # one at a time per invoice
@[Punch::Job::Concurrency(2, group: "stripe")]   # two at a time, shared by every job in the group
```

A job that is due while its key is at the limit waits as `blocked`, out of the way of other jobs, and becomes `ready` again when a job of its key finishes.

## Changing a payload

A payload written by one release is read by another. After a deploy, the new binary reads jobs the old one enqueued, and delayed or retried jobs can outlive several releases. After a rollback, the old binary may meet jobs the new one wrote. Punch keeps both directions working in three layers.

### The payload lock

`payloads.lock.json`, committed beside `openapi.json`, records every job type, its payload version, and the payload's schema. `punch jobs:lock` writes it, and `punch check` fails when it is stale, or when a change since the main branch is one that waiting jobs cannot follow:

| Change | |
|---|---|
| Add a field that is nilable or has a default | Compatible: old payloads get the default; old binaries ignore the field |
| Remove a field that was nilable or had a default | Compatible |
| Add a required field, remove one, or change a field's type | Breaks |
| Remove a job type | Refused until no job of it is waiting anywhere; then `punch jobs:lock --drop 'Type'` |

Most changes can be made compatible: give the new field a default.

### A new version for a breaking change

When a change cannot be compatible, raise the payload's version and keep the old shape as a frozen type that converts itself. The examples app's reminders used to be all alike. When a firmer, final reminder was added, `Remind` gained a required `reminder` field, and version 1 stayed readable:

```crystal source=docs/guides/examples/app/billing/invoices/jobs/remind.cr select=Billing::Invoices::Remind::V1
# app/billing/invoices/jobs/remind.cr
module Billing::Invoices
  # Mails the customer of one overdue invoice a reminder, and records
  # when. Safe to run again: an invoice that was paid, or reminded in the
  # meantime, is skipped. At most two run at a time, so the reminders of
  # a big sweep leave the mail relay's other connections to sign-in mail.
  @[Punch::Job::Options(queue: "mail", attempts: 5)]
  @[Punch::Job::Concurrency(2)]
  @[Punch::Payloads::Version(2, from: {1 => V1})]
  struct Remind
    # ...

    # Version 1: before there was a final reminder, every reminder was a
    # first one. Never change it.
    struct V1
      include Punch::Payloads::Frozen

      getter invoice_id : ID

      def upgrade : Remind
        Remind.new(invoice_id, Reminder::First)
      end
    end
  end
end
```

- `@[Punch::Payloads::Version(2, from: {1 => V1})]` says the current version is 2, and version 1 payloads are read as `V1`.
- `V1` holds the old fields exactly as they were, includes `Punch::Payloads::Frozen`, and never changes. `punch jobs:lock` compares it with what the lock recorded for version 1.
- `upgrade` returns the job: a version 1 reminder was always a first one. A chain such as `{1 => V1, 2 => V2}` reads each old version one step at a time.
- New jobs are written as version 2.

Delete `V1` once `punch jobs:versions` shows no waiting job of version 1 in any environment. Until then, a binary without `V1` would park those jobs.

### Nothing is lost for being unreadable

A worker only claims payload versions its binary reads, so an old binary leaves a newer one's jobs waiting during a deploy or after a rollback. A payload that still fails to parse is parked as `incompatible`, with the error stored: the attempt does not count, and the job waits for `punch jobs:retry` with a binary that reads it, or for `punch jobs:discard`.

## Workers

`serve` runs the workers in its own process by default, in an execution context of their own, so jobs do not take threads from requests. Settings, all optional:

| Setting | Default | |
|---|---|---|
| `PUNCH_JOBS` | `on` | `off` runs no workers in `serve` |
| `PUNCH_JOB_THREADS` | `2` | Threads for jobs |
| `PUNCH_JOB_CONCURRENCY` | `10` | Jobs at once in the process |
| `PUNCH_JOB_QUEUES` | every queue | A comma-separated list to run only some |

To run jobs apart from the web server, start `bin/<app> jobs` beside `serve` with `PUNCH_JOBS=off`. Both use the same `queue.db`, and SQLite's lock makes sure each job is claimed once.

On shutdown, workers stop claiming, wait up to 25 seconds for running jobs, and hand the rest back to the queue without counting the attempt. A process that dies without shutting down stops its heartbeat, and another process hands its jobs back within about a minute.

## Inspecting and fixing jobs

The `punch jobs:*` commands read `queue.db` in `PUNCH_STORAGE`, also while the app runs, and take `--json`:

```console
$ punch jobs:stats
$ punch jobs:list --state failed
$ punch jobs:show 1042
$ punch jobs:retry 1042
$ punch jobs:discard 1042
$ punch jobs:pause mail
$ punch jobs:resume mail
```

`jobs:stats` counts jobs per queue and state and lists the processes with their last heartbeat. `jobs:show` prints one job's payload and its last error with the backtrace. `jobs:retry` gives a failed, discarded, or parked job fresh attempts. `jobs:pause` stops every process from claiming jobs of a queue, while enqueuing goes on. The ops console at `/_punch` shows job runs, their errors, and how long each job type takes ([Ops data](../ops.md)).

A job is in one of these states: `ready` (shown as `scheduled` until it is due), `running`, `blocked`, `done`, `discarded`, `failed` (out of attempts, kept until retried or discarded), and `incompatible` (parked). Finished jobs are purged after 72 hours (`PUNCH_JOB_RETENTION_HOURS`).

## Testing jobs

In specs, nothing runs by itself. Each example gets its own empty queue, and `Punch::Spec.perform_jobs` runs the jobs that are due, follow-ups included, and returns each run's type, outcome, and error. A manual clock, `Punch::Spec.job_clock`, decides what is due.

The examples app's spec runs the same reminder twice:

```crystal source=docs/guides/examples/spec/billing/invoices/jobs/remind_spec.cr select="Billing::Invoices::Remind mails one reminder, also when it runs twice"
# spec/billing/invoices/jobs/remind_spec.cr
describe Billing::Invoices::Remind do
  it "mails one reminder, also when it runs twice" do
    clock = Punch::Spec.job_clock
    invoice = BillingSpec.invoice(due_at: clock.now - 3.days)
    2.times { Punch::Job::Queue.enqueue(Billing::Invoices::Remind.new(invoice.id, :first)) }
    performed = Punch::Spec.perform_jobs
    performed.map(&.outcome).should eq([Punch::Job::Queue::Outcome::Done, Punch::Job::Queue::Outcome::Discarded])
    performed.last.error.should eq("invoice #{invoice.id} was reminded already")
    Punch::Spec.mails.map(&.subject).should eq(["Invoice #{invoice.number} is overdue"])
  end

  # ...
end
```

The reference app's spec follows a job from the Command that enqueues it. The `enqueue` matcher checks what a block put in the queue, the outbox included, and the clock moves the reminder's due date into reach:

```crystal source=reference/spec/billing/invoices/remind_spec.cr select="invoices API reminds once when a sent invoice comes due, through the outbox"
# spec/billing/invoices/remind_spec.cr
# ...

describe "invoices API" do
  it "reminds once when a sent invoice comes due, through the outbox" do
    clock = Punch::Spec.job_clock
    client = AccountsSpec.client
    customer = customer_id(client)
    id = draft(client, customer)
    invoice_id = id
    send = -> { client.post(Billing::Invoices::Send.path(invoice_id: invoice_id), headers: Punch::Spec.idempotency_key).status.should eq(HTTP::Status::OK) }
    send.should enqueue(Billing::Invoices::Remind, invoice_id: invoice_id, times: 1)
    # Only the mail runs now; the reminder is not due for 14 days.
    Punch::Spec.perform_jobs.map(&.type).should eq(["Billing::Invoices::MailInvoice(Billing::InvoiceSent)"])
    clock.advance(Billing::Invoices::PAYMENT_TERM + 1.minute)
    Punch::Spec.perform_jobs.map(&.outcome).should eq([Punch::Job::Queue::Outcome::Done])
    reminded = Punch::Spec.db.read { |db| Billing::Invoices::Queries.reminded(db, invoice_id: invoice_id) }
    (reminded.try(&.created_at) || fail "no reminder").should be_close(clock.now, 1.second)
    # A job that runs again writes no second reminder.
    job = Billing::Invoices::Remind.new(invoice_id)
    Punch::Job::Queue.enqueue(job)
    Punch::Spec.perform_jobs.map(&.outcome).should eq([Punch::Job::Queue::Outcome::Done])
    Punch::Spec.db.read { |db| db.query_all("SELECT count(*) FROM invoice_reminders", &.read(Int64)) }.should eq([1])
  end

  # ...
end
```

A payload conversion is tested by reading an old payload as a worker would, with `Punch::Spec.read_payload`:

```crystal source=docs/guides/examples/spec/billing/invoices/jobs/remind_spec.cr select="Billing::Invoices::Remind reads a payload of version 1 as a first reminder"
# spec/billing/invoices/jobs/remind_spec.cr
describe Billing::Invoices::Remind do
  # ...

  it "reads a payload of version 1 as a first reminder" do
    remind = Punch::Spec.read_payload(Billing::Invoices::Remind, 1, %({"invoice_id": 7}))
    {remind.invoice_id.value, remind.reminder}.should eq({7, Billing::Invoices::Reminder::First})
  end

  # ...
end
```

To call a Command directly in a spec, get its outbox handle with `Punch::Spec.jobs(Command, writer)` ([Background jobs: specs](../jobs.md#specs)).

## Common mistakes

**Calling another system from a Command.** A Command that mails or calls a provider in `call` holds the database's write lock while it waits, and the call cannot be undone if the transaction rolls back. Enqueue a job through `jobs : Punch::Jobs(…)` instead.

**Enqueuing directly inside a transaction.** `Punch::Job::Queue.enqueue` in a Command, or inside `job.write`, raises:

```text
Billing::Invoices::Remind enqueued directly inside a write transaction. A Command enqueues through its outbox: `jobs : Punch::Jobs(Billing::Invoices::Remind)` in `call` (lib/punch/docs/jobs.md); other code enqueues after the write has committed.
```

Ask for `Punch::Jobs` in the Command's `call`, or move `job.enqueue` after the `job.write` block.

**A job that is not safe to repeat.** A job that charges a card or sends a mail without first checking whether that already happened does it twice after a crash. Check the state, record what was done, and pass `job.key` to providers that take one.

**A whole record in the payload.** The job then works on the row as it was when enqueued. Store the ID and read the row in `perform`; a field holding a record does not compile.

**A recurring job with a required field.** Its payload is empty, so the field could never be set:

```crystal fails=spec/fail/recurring_required_field.cr
@[Punch::Job::Recurring(every: 1.hour)]
struct Report
  include Punch::Job

  getter account_id : Int64

  def initialize(@account_id)
  end

  def perform(job : Punch::Job::Context) : Punch::Job::Result
  end
end

Punch::Job::Queue.tasks
```

```text
Error: Report is recurring, so it runs with an empty payload; give Report#account_id a default or make it nilable (lib/punch/docs/jobs.md).
```

**A job as a class.** A job is a value stored in `queue.db`, so it is a struct. `class Export` with `include Punch::Job` does not compile.

**Adding a required field and refreshing the lock.** `punch check` fails on the `payloads` step, and running `punch jobs:lock` to make it pass does not help: the lock refuses a breaking change too. Give the field a default, or raise the version with a frozen type.

**Deleting a frozen type early.** Jobs of that version that are still waiting in production are parked as `incompatible`. Check `punch jobs:versions` in every environment first.

## Limits

- Jobs run at least once, never exactly once. Repeating safely is the job's part.
- Schedules are in UTC, and runs missed while no process was up are not made up.
- The queue is a SQLite file on the app's machine. Several processes on that machine share it; processes on other machines cannot.
- A job's priority orders jobs that are due; it does not take a worker from a job that is already running.

## Rules

- Never call another system from a Command's `call`; enqueue a job. ([Enqueuing from a Command: the outbox](#enqueuing-from-a-command-the-outbox))
- Enqueue from a Command only through `jobs : Punch::Jobs(…)`; use `Punch::Job::Queue.enqueue` and `job.enqueue` only outside a write transaction. ([Enqueuing outside a transaction](#enqueuing-outside-a-transaction))
- Make every job safe to run twice: check the state first, and pass `job.key` as the idempotency key where a provider takes one. ([Jobs run at least once](#jobs-run-at-least-once))
- Keep payloads to IDs and small values; read rows when the job runs. ([What a job is](#what-a-job-is))
- Return `retry(reason)` or `discard(reason)` for expected outcomes; raise only for bugs. ([What a job is](#what-a-job-is))
- Split a sweep into one job per item. ([Running on a schedule](#running-on-a-schedule))
- Never make a payload change that breaks waiting jobs: add fields with defaults, or raise the version with a frozen type, and keep the frozen type until `punch jobs:versions` shows none left. ([Changing a payload](#changing-a-payload))
- Commit `payloads.lock.json`, and never edit it by hand. ([The payload lock](#the-payload-lock))

## Reference

- [Background jobs](../jobs.md): every option, the outbox table, the workers' internals, states, commands, tracing, and spec helpers.
- [Domain events](../events.md): deferred subscribers, which run as jobs.
- [Mail](../mail.md): delivering mail from a job.
- [Ops data](../ops.md): job runs in the ops console.
