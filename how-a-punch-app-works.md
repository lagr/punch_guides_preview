# How a Punch App Works

Applies to Punch 0.1.

Before the topic guides go deep into one part each, this guide draws the map. It follows one request from the socket to the response and on into the background, names the files an app keeps its data in, and says which binary does what. Every later guide fills in one region of this map, and each section here links to it.

After reading this guide, you will know:

- the path of a request through Punch, and where your code plugs in;
- why Queries get a `Reader` and Commands a `Writer`, and what that buys you;
- what happens after a Command commits: subscribers, the outbox, and jobs;
- which files hold an app's data, and which files are generated and never edited by hand;
- what the `punch` tool does and what the app's own binary does;
- where each of the next guides fits on the map.

Before you start: you have read the [Introduction](introduction.md). The examples come from the reference app, an invoicing app in `reference/`.

## One request, end to end

The clearest way to see how the parts fit is to follow one request. Here a program sends an invoice through the reference app's JSON API, with an API token:

```console
$ curl -X POST http://localhost:3000/billing/invoices/1/send \
    -H "Authorization: Bearer inv_…" -H "Idempotency-Key: 6f1c2a"
{"id":1,"status":"sent","number":"INV-000001","total":30000,"due_at":"2026-11-06T19:40:53Z",…}
```

The Command that answered is this one:

```crystal source=reference/app/billing/invoices/operations/send.cr select=Billing::Invoices::Send
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
    struct Input
      include Punch::Input

      @[Punch::PathParam]
      getter invoice_id : ID
    end

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

Nothing in this file mentions HTTP. The annotations say who may run it, how often, and that a client must name each attempt. The input says what the request must carry. `call` says what it needs (a `Writer`, the actor, a grant from the policy, an event handle, a job handle, the clock) and what can come of it (`Output`). Punch derives everything else: the route, the parsing, the checks, the transaction, the status codes, the OpenAPI entry.

With `PUNCH_TRACE=console`, which `punch dev` sets for you, the app prints each request's trace as a tree. This is the trace of the request above, as the reference app printed it:

```console
POST /billing/invoices/:invoice_id/send 200 detail  1.25 ms  queries 17  trace 8d5fa940f00c192a29e7e1fb163583fd
    +0.07    0.05 ms  input
    +0.13    0.78 ms  primary write wait_ms=0.049
    +0.21    0.02 ms    Punch::Auth::APITokens.find rows=1
    +0.24    0.01 ms    Accounts::Users::Table.find rows=1
    +0.26    0.01 ms    Punch::Roles.held rows=2
    +0.28    0.00 ms    Accounts::Organizations::Queries.choice_by_token rows=1
    +0.30    0.01 ms    Accounts::Organizations::Table.find rows=1
    +0.33    0.02 ms    Billing::Invoices::Queries.organization rows=1
    +0.37    0.03 ms    Punch::IdempotencyKeys.find rows=1
    +0.41    0.02 ms    Billing::Invoices::Table.find rows=1
    +0.44    0.02 ms    Billing::Invoices::Queries.lines rows=1
    +0.48    0.04 ms    Billing::Invoices::Queries.last_sequence rows=1
    +0.54    0.03 ms    Billing::Invoices::Queries.save changes=1
    +0.58    0.03 ms    Billing::Invoices::Queries.delete_lines changes=1
    +0.61    0.03 ms    Billing::Invoices::Lines::Table.insert rows=1
    +0.64    0.05 ms    publish Billing::InvoiceSent
    +0.66    0.03 ms    Punch::Job::Outbox.add changes=1
    +0.71    0.01 ms    Punch::Job::Outbox.add changes=1
    +0.76    0.05 ms    Punch::IdempotencyKeys.store changes=1
    +0.92    0.22 ms  Billing::Customers::ForgetStatement
    +0.94    0.21 ms    cache write wait_ms=0.043
    +0.99    0.02 ms      Punch::Caches::Durable.delete changes=0
job Billing::Invoices::MailInvoice(Billing::InvoiceSent)  0.66 ms  queries 5  trace 8d5fa940f00c192a29e7e1fb163583fd
    +0.02    0.22 ms  primary read
    +0.03    0.07 ms    Billing::Customers::Table.find rows=1
    +0.10    0.03 ms    Billing::Invoices::Table.find rows=1
    +0.14    0.03 ms    Billing::Invoices::Queries.lines rows=1
    +0.18    0.02 ms    Billing::Invoices::Queries.payment rows=1
    +0.20    0.03 ms    Accounts::Organizations::Table.find rows=1
    +0.36    0.19 ms  mail.deliver
```

The rest of this guide walks down that tree.

## Routing: the name is the address

The request is `POST /billing/invoices/1/send`. Punch matched it to `Billing::Invoices::Send` without a routes file. A route is derived from the operation's full name and its input, by three rules: the namespace becomes the path (`Billing::Invoices` is `/billing/invoices`), a path parameter becomes a segment (`invoice_id` is `/:invoice_id`), and the last part of the name becomes the method and the action (`Send` is `POST …/send`, while the reserved names `Index`, `Show`, `Create`, `Update`, and `Destroy` map to the plain REST routes). A Query answers `GET`; a Command answers `POST`, `PATCH`, or `DELETE`.

The same name is also the operation's `operationId` in `openapi.json`, its permission ID (`billing.invoices.send`), its name in traces, and the file it lives in. One name, read anywhere, leads to one file.

`punch routes` lists what Punch derived. Details and the rules for overriding a route, which you rarely do, are in [Operations and Routing](operations-and-routing.md).

## Three surfaces from one set of operations

The trace says `200 detail`: the request asked for JSON, and the result type `Detail` was rendered as JSON. The same app also serves people and agents, from the same operations:

- **JSON for programs.** Every Query and Command is an endpoint. Its OpenAPI entry is derived from its input and its `Output`, and committed as `openapi.json`.
- **HTML for people.** A page (`Punch::Page`, named `…Page`) calls operations and renders components. A form on a page is bound to a Command, so a submitted form runs the same Command as the API.
- **Markdown for agents.** A page marked `@[Punch::AgentReadable]` also answers `Accept: text/markdown`, with its forms as action blocks an agent can follow.

A page and a Query can share a path. `GET /billing/invoices` answers a browser with the invoice list page and a program with JSON, chosen by the `Accept` header. Pages are in [Operations and Routing](operations-and-routing.md#a-browser-and-a-program-on-one-path); the HTML and markdown surfaces are in the [views reference](../views.md).

## Before `call`: parsing, then checks

Punch does several things before the operation's code runs. Each one can answer the request itself, and when it does, `call` never runs.

1. **Rate limits by address.** A limit `by: :ip` is checked first, before any work (429 when over).
2. **Input.** The `input` span. Path parameters, the query string, a form, or a JSON body are parsed into `Send::Input`. A field that does not parse is a rejection, and the client gets 422 with every error at once, each at its field. Inside `call`, `input.invoice_id` is already a typed `Billing::Invoices::ID`. See [Input and Domain Types](input-and-domain-types.md).
3. **The transaction opens.** The `primary write` span. Everything below happens inside one transaction, so the checks and the work see the same data.
4. **The actor.** The token is looked up (`APITokens.find`), then the user and the organization the token works in. The result is the app's actor type, `Invoicing::Actor`. No valid token or session means 401 for an operation that requires an actor. See [Authentication](authentication.md).
5. **Rate limits by actor.** `Send` allows 30 an hour per user.
6. **The policy.** `@[Punch::Authorize(ByRole)]` asks the slice's policy whether this actor may send this invoice (`Invoices::Queries.organization` reads the invoice's organization, `Punch::Roles.held` the actor's roles). A refusal is 404 when the actor may not even see the invoice, 403 when they may see it but not do this. A permission comes back as a `Punch::Grant(Send)`, which `call` asks for, so a Command cannot run without having been checked. See [Authorization and Roles](authorization-and-roles.md).
7. **The idempotency key.** `IdempotencyKeys.find` looks for an earlier attempt with the same key. If one finished, its stored answer is replayed and `call` does not run again.

None of this is optional, and none of it is middleware you remember to add. An operation without `@[Punch::Authorize(...)]` or `@[Punch::Public]` does not compile.

> [!NOTE]
> **Coming from Rails:** where a controller uses `before_action` filters, an operation declares the same things on itself, and Punch derives them from its type: the actor from the `call` parameter, the permission from the policy annotation, the limit from its annotation. See [Coming from Rails](coming-from-rails.md).

## `call`: a Reader or a Writer

`call` gets a handle to the database, and the handle's type decides what it may do:

- A **Query** gets a `Punch::Reader`. It runs in a read transaction on one of a pool of read-only connections, many at once, each seeing one consistent snapshot. A Query cannot write; that does not compile.
- A **Command** gets a `Punch::Writer`. Each SQLite file has one writer connection, and write transactions queue for it and run one after another, so writes inside one process never fail with "database is locked". `wait_ms` in the trace is how long this one waited in that queue.

So a Command is one transaction. Everything `Send` does (loading the draft, taking the next number, saving the sent invoice, publishing the event, enqueuing the reminder) commits together or not at all.

`call` returns one of its `Output` types. The type decides the response: its `@[Punch::Status(...)]` gives the status (200 by default), a type that includes `Punch::Problem` is a failure with a status from 400 on and is rendered as Problem Details. **A Command whose result is a failure is rolled back.** If `Send` returned `NoLines`, nothing it wrote would stay, without a `rescue` or a manual rollback. See [Results, Errors, and Status Codes](results-errors-and-status-codes.md).

The rest of `call` is your code, and most of it is not in the operation. The reference app's `Send` loads the invoice from its store, asks the model whether it may be sent (`invoice.send(...)` returns a sent invoice or `NoLines`), saves the result, and reports what happened. The rules live in the model, `Billing::Invoices::Invoice`; the SQL lives in named queries (`Billing::Invoices::Queries.save` in the trace), generated from `.sql` files into typed methods. A model never touches the database; an operation orchestrates. See the [models reference](../models.md) and the [queries reference](../queries.md).

## After the commit: events, the outbox, and jobs

Look at the end of the write in the trace:

```text
publish Billing::InvoiceSent
Punch::Job::Outbox.add changes=1
Punch::Job::Outbox.add changes=1
Punch::IdempotencyKeys.store changes=1
```

`events.publish(InvoiceSent…)` records a fact, and `jobs.enqueue(Remind…)` asks for work later. Neither runs anything yet. Both are written into `punch_outbox`, a table in `primary.db`, inside the Command's own transaction: one row for the reminder, one for the subscriber that mails the invoice. If the Command rolls back, they are gone with its other writes. If it commits, they will run. That is what makes "the invoice is sent, and the customer gets it" reliable without a distributed transaction.

Then the transaction commits, and three kinds of reaction follow:

- **After-commit subscribers** run in the request, right after the commit and before the response. `ForgetStatement` forgets the customer's cached statement, so the next read computes it fresh. A failure there is recorded but does not change the response.
- **Deferred subscribers** run as jobs. `MailInvoice` subscribes to `InvoiceSent` with `:deferred` delivery, because sending mail calls out of the app and may fail and retry. It ran a moment after the response, as the last part of the trace shows (`job Billing::Invoices::MailInvoice(Billing::InvoiceSent)`), under the same trace ID.
- **Jobs** enqueued by the Command run when they are due. `Remind` waits until the invoice's due date.

The outbox relay moves committed rows into `queue.db`, where workers claim them. Workers run inside `serve` by default, in their own execution context, so jobs never take threads from requests. A job that fails is retried with backoff; one that keeps failing ends as `failed` and appears in the ops console.

See [Domain Events](domain-events.md) and [Background Jobs](background-jobs.md).

## The response

Last comes the response. The idempotency key's answer is stored (`IdempotencyKeys.store`) in the same transaction, so a retry with the same key replays this exact response. The result is rendered with the status from its type, and Punch adds the headers every response gets: a Content-Security-Policy, `X-Content-Type-Options`, `Cache-Control: no-store` unless the operation set its own, and `X-Request-ID`, the trace ID, which every error body carries too.

When the request ends, the trace becomes one **wide event**: one JSON line per request with the route, the operation, the outcome (`detail`), the status, who asked, and totals for every statement. In production it goes to stdout and, if configured, to an OpenTelemetry collector. Failed and slow requests are also stored in `ops.db` for the ops console. See [Tracing and Metrics](tracing-and-metrics.md).

## The storage directory

All of an app's data is a few SQLite files in one directory, `PUNCH_STORAGE`, on local disk:

| File | Holds | Whose |
|---|---|---|
| `primary.db` | The app's tables, plus Punch's tables the app's migrations create (sessions and tokens, roles, the outbox, idempotency keys, webhook deliveries) | Yours: your migrations, your named queries |
| `queue.db` | Jobs: waiting, running, finished, failed; recurring schedules | Punch's |
| `ops.db` | Error groups, stored wide events, committed events, performance per route and query | Punch's |
| `cache.db` | Values of the app's caches | Punch's; may be deleted at any time |

You change `primary.db` only through migrations and your own SQL. The other three are Punch's, with layouts Punch manages; never write to them. Backups with Litestream replicate all four. Databases on a network filesystem are refused at startup, because SQLite's locking needs local disk. See [SQLite in Punch](sqlite-in-punch.md).

## Two binaries

You work with two programs:

- **`punch`**, the standalone tool, built from the Punch version your app depends on. It writes and checks files: `punch new`, `punch generate`, `punch db:migration`, `punch db:generate`, `punch i18n:generate`, `punch openapi`, `punch check`, `punch dev`, `punch typecheck`, `punch routes`.
- **The app's own binary**, `bin/invoicing` for the reference app, built from your code. It runs and inspects the app: `serve`, `jobs`, `db:migrate`, `db:audit`, `tokens:create`, `roles:assign`, `config`, `ops`.

You rarely need to tell them apart. `punch` forwards a command it does not have to the app's binary, so `punch tokens:create` builds the app and runs `bin/invoicing tokens:create`. In production only the app's binary exists: one static file with the migrations, assets, translations, and `openapi.json` embedded. At boot it checks its settings and credentials, applies pending migrations, and starts serving. See [Configuring Punch Applications](configuring-punch-applications.md) and [Deploying](deploying.md).

## Generated files

Several files in an app are written by tools from a source you edit. They are committed, so a reviewer sees what changed, and `punch check` fails when one is stale. Never edit them by hand:

| File | Source | Written by |
|---|---|---|
| `queries.gen.cr`, `records.gen.cr` | `queries.sql`, `columns.cr`, `db/schema.sql` | `punch db:generate` |
| `db/schema.sql` | the migrations in `db/migrations/` | `punch db:migrate` |
| `app/translations.gen.cr` | `config/locales/*.yml` | `punch i18n:generate` |
| `app/requires.gen.cr` | the files under `app/` | `punch requires:generate` |
| `openapi.json` | the operations | `punch openapi` |
| `payloads.lock.json` | job and event payload types | `punch jobs:lock` |

`punch dev` and `punch typecheck` regenerate the first four before each build. See [`punch check` and the Command Line](punch-check-and-the-command-line.md).

## What keeps it true

Three layers of checks stand behind every rule in these guides, from fastest to slowest:

1. **The compiler.** A missing policy, a Query that writes, an unhandled result in a `case`, a link to a page with the wrong parameter, a secret in a JSON response: these do not compile. Most messages say what to do and name the reference page. See [When the Compiler Says No](when-the-compiler-says-no.md).
2. **Ameba with Punch's rules.** Conventions the compiler cannot see, such as a file in the wrong place or a status compared inside an operation.
3. **`punch check`.** Everything CI runs, the same on a laptop: formatting, lint, the build, stale generated files, query plans that scan whole tables, migrations that rebuild a table without saying so, breaking API changes against the main branch, breaking job payload changes, and the specs.

A coding agent and a person work against the same checks. When a check fails, the honest fix is to change the code, not the check.

## Where to go next

| To learn | Read |
|---|---|
| Where each kind of file goes | [Application Structure](application-structure.md) |
| Routes and what `call` can ask for | [Operations and Routing](operations-and-routing.md) |
| Inputs and domain types | [Input and Domain Types](input-and-domain-types.md) |
| Outcomes and status codes | [Results, Errors, and Status Codes](results-errors-and-status-codes.md) |
| The JSON API and `openapi.json` | [Building a JSON API](building-a-json-api.md) |
| Tables, SQL, and models | [SQLite in Punch](sqlite-in-punch.md), [Named Queries](named-queries.md), [Domain Models](domain-models.md) |
| Pages and forms | [Pages and Components](pages-and-components.md), [Forms](forms.md) |
| Events and jobs | [Domain Events](domain-events.md), [Background Jobs](background-jobs.md) |
| Specs | [Testing Punch Applications](testing-punch-applications.md) |

## Common mistakes

**Calling another system from `call`.** A Command posts to a payment provider or sends mail directly. If the transaction then rolls back, the call cannot be taken back; if the call fails, the request fails with it. Enqueue a job or publish an event with a deferred subscriber, so the call happens after the commit and is retried on its own.

**Expecting a Query to write "just this once".** A Query that asks for a `Punch::Writer` does not compile:

```crystal fails=spec/fail/query_with_writer.cr
module Invoices
  @[Punch::Public]
  struct Purge < Punch::Query
    struct Input
      include Punch::Input
    end

    def call(input : Input, db : Punch::Writer) : Nil
    end
  end
end

Punch::Routing::Router.new(Punch::Database.memory)
```

```text
Error: Invoices::Purge is a Query, so its `call` receives a Punch::Reader, not Punch::Writer (lib/punch/docs/operations.md).
```

A read that has to record something, such as a "last viewed" time, is a Command, or a job enqueued from one.

**Reading `queue.db` or `ops.db` from app SQL.** They are Punch's, with layouts that change between versions. Use `punch jobs`, `punch ops`, or the ops console.

**Editing a generated file.** The next `punch dev` overwrites the edit, and `punch check` fails on it before that. Change the source in the table above and run its generator.

**Putting the storage directory on a shared folder.** A Docker volume on macOS, an NFS mount, or a synced folder fails the filesystem check at startup, or worse, corrupts data where the check cannot see it. Keep `PUNCH_STORAGE` on local disk.

## Limits

- One machine. Punch apps run on one node; the job workers can run in a separate process on the same machine, but there is no cluster.
- Live page updates reach only browsers connected to the process that committed the change. A separate `jobs` process does not reach them.
- No HTTP/2 or TLS in the app. A reverse proxy (kamal-proxy, Caddy) in front handles both.

## Rules

- Put each kind of work where the map puts it: the input's types parse, the policy decides who, `call` orchestrates, the model decides, named queries hold the SQL. ([Before `call`: parsing, then checks](#before-call-parsing-then-checks), [`call`: a Reader or a Writer](#call-a-reader-or-a-writer))
- A Query reads and a Command writes; never work around the handle. ([`call`: a Reader or a Writer](#call-a-reader-or-a-writer))
- Return failures as types; never roll back by hand. ([`call`: a Reader or a Writer](#call-a-reader-or-a-writer))
- Call out of the app only from jobs and deferred subscribers, never from `call`. ([After the commit: events, the outbox, and jobs](#after-the-commit-events-the-outbox-and-jobs))
- Touch only `primary.db`, and only through migrations and your named queries. ([The storage directory](#the-storage-directory))
- Never edit a generated file; change its source and run its generator. ([Generated files](#generated-files))

## Reference

- [Operations and routing](../operations.md#from-request-to-response): the request steps in detail.
- [Persistence](../persistence.md): connections, readers, the writer, and the filesystem check.
- [HTTP kernel](../http.md): responses, headers, timeouts, and shutdown.
- [Background jobs](../jobs.md#from-a-command-the-outbox): the outbox and the workers.
- [Ops data](../ops.md): what `ops.db` records.
- [Tooling](../tooling.md): the commands of both binaries.
