# Small Team, Full Punch

Coding agents have made writing code cheap. A single developer can now write a lot of it in a day. What agents have not made cheap is the work around the code: knowing that a change is right, getting it into production without breaking anything, and knowing what the running system is doing. If anything, that work has grown, because there is more code to review and it arrives faster than anyone can read it carefully.

## A dozen services before the first customer

Think about what a small business app needs before it has a single customer. A database server. A cache, and often a second server for the job queue. An error tracker, a log service, something for performance, something for uptime. Object storage for uploads. Somewhere to keep secrets. A backup scheme that, in many projects, nobody has ever restored from. Each of these is a reasonable choice on its own. Together they are a lot: each one has its own dashboard and its own way of failing at 3 a.m.

For a company with a platform team this is fine. Someone owns each part. For one person it means the product is mostly made of parts they either rent and only half understand, or host themselves and now have to patch, back up, and watch, each one another product to run beside your own.

So the question I started from was: if writing code is cheap, what should a framework for a very small team optimise for? My answer is the two things that stay expensive. A change should be easy to verify, and the running system should be easy to read, for the person and for the agent working beside them.

Punch is my answer to that question: a web framework for Crystal, built for one developer, or a team of a few, who write much of their code with coding agents and want to own the whole product, from the first commit to changing it while people use it.

## The basics

First, the boring part, because a framework that is clever about operations and leaves you to build sign-in yourself has its priorities wrong.

`punch new` writes an app that already has accounts: sign-up, sign-in, password reset, email verification, two-factor sign-in with an authenticator app, API tokens, and a list of your own sessions. It has a layout, an ops console (a built-in admin page for errors, jobs, and performance), deploy recipes for Kamal, or for Caddy with systemd on a plain server, a CI workflow, and an `AGENTS.md`. It installs the dependencies, sets up encrypted credentials and signing keys, creates the database, adds a development user, runs every check, and makes the first commit. The first check that ever fails in a new app is about your own change.

From there you get what you would expect: routing, forms that show their errors again, server-rendered pages with Hotwire's Turbo for the parts that update in place, background jobs with retries and schedules, mail, inbound webhooks, direct uploads to S3-compatible storage, a cache, translations, migrations in plain SQL, and a spec library with a test client.

## One machine, a few files

In production, a Punch app is one static binary and a directory with four SQLite files in it: the app's data, the job queue, the operations data, and a cache. Litestream runs beside it and replicates the data to S3-compatible storage within about a second. A reverse proxy, kamal-proxy or Caddy, terminates TLS in front. For an invoicing app I built with Punch to test it, the container image is about 29 MB, built `FROM scratch`: the binary, Litestream, CA certificates, and time zones. There is no shell in it to patch.

There is no Redis, no database server, no separate job service, and no Node toolchain. The job queue is a SQLite file. So are the errors, the traces, and the performance numbers the ops console shows.

This is the trade-off Punch makes most visibly, and it has a cost. A Punch app runs on one machine. There are no read replicas, no multi-writer database, no automatic failover, and it needs a persistent local disk, so stateless container platforms are out. If your product needs any of those, Punch is the wrong tool, and its documentation says so up front.

The bet is that most business software (SaaS for other businesses, internal tools, a JSON API for a mobile app) fits comfortably on one good server, and that for a small team the simplicity of one box is worth more than the ability to grow past it. Machines are big now, and SQLite is fast when it runs in the same process as the code that queries it. A system with four files in it is a system one person can hold in their head.

It also changes what deploying means. A deploy starts the new binary beside the old one, on the same files. The new process applies its migrations and only answers its health check when it is ready; a migration that fails stops it before it gets any traffic. Then the proxy switches over, and the old process finishes its requests and jobs and exits. In benchmarks with the invoicing app, three deploys under load, with about 400,000 requests each, lost none of them, with either proxy. Rolling back is deploying the previous binary.

## Trained by the compiler

Crystal reads a lot like Ruby, but it is compiled and statically typed, and its compiler is strict: unpleasant at first, and later the reason you can trust a change you did not write yourself.

Punch uses that strictness for the mistakes that usually reach production. Business logic in a Punch app is a set of operations: a Query reads, a Command writes. An operation without an access policy does not compile:

```text
Error: Billing::Invoices::Send needs a policy, @[Punch::Authorize(Policy)], or @[Punch::Public]: an operation without either would run for anyone (lib/punch/docs/authorization.md).
```

Neither does a Command that anyone on the internet may call without a rate limit, or one that can return a failure with no HTTP status, which would answer 200 and commit its writes. If an operation can end three ways, its type says so, and every caller has to handle all three. Add a state to a model, and every `case … in` that does not handle it stops compiling.

Each message follows the same pattern: what is wrong, named by your own type; why, in a few words; the fix; and the page of the documentation that explains it, which ships inside the app. An agent that hits a compile error does not need to guess. It reads the message, opens the page, and does the fix the message names.

The other half of this is parsing at the boundary. Everything that enters the app, from a request, a database row, a job payload, or a credential, is parsed once into a type that can only hold valid values. Inside an operation there is no string that should have been an email address. There is an `Email`, or there is a refusal that went back to the form before your code ran.

## One operation

Here is what that looks like in practice: the Command that sends an invoice in the invoicing app, complete.

```crystal
# app/billing/invoices/operations/send.cr
module Billing::Invoices
  # An invoice without lines cannot be sent.
  @[Punch::Status(422)]
  struct NoLines
    include Punch::Problem
  end

  # Sends a draft: it gets the next invoice number and is due after the
  # payment term, and it keeps the customer's name and address as they
  # are now. Only a draft with lines can be sent. Each user sends at most
  # 30 an hour, since each one mails a customer (`MailInvoice`). A client
  # must name each attempt with an idempotency key, so a retry after a
  # lost answer gets that answer instead of `already_sent`.
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

        # The customer is there: the foreign key keeps it while an invoice is.
        customer = Customers::Table.find(db, invoice.customer_id) || return NotFound.new
        case sent = invoice.send(Store.next_number(db), now, BilledTo.of(customer))
        in Invoice::NoLines then NoLines.new
        in Invoice::Sent
          Store.save(db, sent)
          events.publish(InvoiceSent.new(sent.id, sent.customer_id))
          # Through the outbox: the reminder exists exactly when the sent invoice does.
          jobs.enqueue(Remind.new(sent.id), at: sent.due_at)
          Store.detail(db, sent, now)
        end
      end
    end
  end
end
```

The comment above `Send` is its description, for the people who read the code and in the API documentation. The three annotations below it are the policy, the rate limit, and the idempotency key it describes. The input is one typed ID from the path. The `Output` line lists every way the call can end: the invoice, not found, already sent, or no lines, and `NoLines` carries its own HTTP status.

The arguments of `call` are everything it may touch: a database handle that can write, the user, proof that the policy allowed this call, an event stream that can only publish `InvoiceSent`, a job queue that can only take `Remind`, and the clock. Nothing else is in reach.

The body is a `case … in` over the invoice's states. A draft, a sent invoice, and a paid one are each their own type, and only a draft has a `send` method. Its answer is again a choice of types, a sent invoice or `NoLines`, and the second `case` has to handle both. If invoices gain a `Cancelled` state next year, this file stops compiling until it says what sending a cancelled invoice does. The reminder job, and the mail the event triggers, are written in the same transaction that saves the invoice, so the reminder exists exactly when the sent invoice does.

Apart from the status on `NoLines` and the path parameter, nothing in it mentions HTTP. The rest of this post is mostly about what Punch makes of these fifty lines.

## One declaration, three readers

A Query receives a database handle that cannot write, so a Query cannot change data, even by mistake. Otherwise it looks like `Send`: a typed input, a typed set of outcomes, and nothing about HTTP.

From that one declaration Punch serves three surfaces. A page beside the operation renders HTML for people. The same page, asked for as markdown, renders for agents. And the operation answers JSON for programs, documented in an OpenAPI file that is derived from the types and committed to the repository. The route, the input parser, the form fields, the error codes, and the API schema all come from the same types, so they cannot drift apart.

`Send` becomes `POST /billing/invoices/{invoice_id}/send`, a button on a draft's page, and an operation in the OpenAPI file. Here is part of an invoice draft as the invoicing app answers it to an agent:

```text
# Draft inv_GccxS3jabFg

- Customer: Ada Lovelace
- Status: draft

## Actions

### Send invoice

`POST /billing/invoices/inv_GccxS3jabFg/send.md` · operation `billing.invoices.send` · not idempotent · at most 30 per 1 h per user

- `_idempotency_key` = `766i5UBC5-S7LRnKAJx1Cgh_` (send it as it is, also when you retry)

### Delete draft

`DELETE /billing/invoices/inv_GccxS3jabFg.md` · operation `billing.invoices.destroy` · idempotent · destructive
```

Nobody wrote that markdown. The rate limit and the idempotency key are `Send`'s annotations, read back. The page comes from the same components that wrote the HTML page, with the navigation left out and each form turned into an action block. An action shows up only where its button would: the Send block is on a draft, not on a sent invoice, and not for a user whose role may not send. The agent reads the same app a person sees, with the same permissions, and there is no second API to keep in step.

## Written for agents, checked for everyone

Most of what makes a codebase pleasant for an agent also makes it pleasant for a person. Punch's conventions are few, written down, and checked.

A type's name decides its file, its route, and its permission, so there is one obvious place for each thing. Queries are SQL files, pages are written with HTML element names, the API contract is OpenAPI: languages people and models already know. Generated code, such as the typed query results, the translations, the OpenAPI document, and a lock file recording the shape of every job's payload, is committed, so a reviewer sees what changed. Every app has an `AGENTS.md` that states the working loop:

1. Edit, then `punch typecheck`, which checks the whole app in a few seconds without building it (`--json` gives the errors as data).
2. Run the specs of the area you changed.
3. Look at it in `punch dev` if it shows on a page.
4. Run `punch check` before you commit.

`punch check` runs everything CI runs: the formatter, the schema against the migrations, the generated files, the lint rules, the query plans (a full table scan is reported), the build, the policy and rate-limit checks, the job payloads and the OpenAPI document against the main branch, and the specs. It keeps going after a failure, so one run reports everything, and each failure says what to do.

Punch's documentation calls this "the honest fix". When a check fails, the fix is never to edit a generated file, loosen a check, or switch a rule off without a reason a reviewer would accept. Agents, like tired people, are tempted to make the red line go away. A framework can make the honest fix the easy one: say exactly what it is, and make the dishonest ones visible in review.

## Running it, and changing it while it runs

So far that covers starting and building. The other half of owning a product happens after the first deploy, and Punch keeps that inside the app as well.

The ops console at `/_punch` shows errors grouped by cause. Each occurrence comes with its wide event, the one record Punch keeps per request or job: the route, the input, the outcome, and every named SQL query it ran. The console also shows failed jobs with their payloads, performance per route, job, and query, the events Commands published and who handled them, the database files, and how far the backup is behind. Backups come with a restore drill, so you can practise losing the machine before it happens. Trouble is announced as events, `ErrorOccurred`, `JobsFailing`, `BackupStale`, and alerting is a subscriber in your own app that mails you.

Changing a live app is where small products usually get hurt, because now there is data. Punch's rule is that a release works with the schema, the stored data, the waiting jobs, the signing keys, and the API clients of the release before it, because for a few seconds during every deploy both run against the same files. `punch check` compares your branch with main and catches many of the changes that would break that: a job payload the old release cannot read, an API change that breaks a correct client, a migration that rebuilds a table without saying so. It cannot see which columns the previous release still reads, so removing or renaming a column is yours to stage. Punch's documentation walks through each kind of change, including how to split one across releases.

## The night shift

The ops console has a markdown surface for a reason. Every console page is readable by an agent. With an operator's API token, `/_punch/errors.md` is a list of open error groups, each linking to its backtrace and the wide events of its latest occurrences. `/_punch/jobs.md` lists failed jobs by type with their latest error. `/_punch/performance.md` lists the slowest routes and queries. `/_punch/database.md` says whether the replica is behind. `punch ops --json` gives the same from the command line, on the server or against a running app.

That is enough for a loop you can run with any coding agent. It wakes up on a schedule and reads the overview. If a new error group opened since its last look, it opens the group, follows the stored wide event to the operation and the query, reads the code, writes a spec that reproduces the failure, writes a fix, runs `punch check`, and opens a pull request with what it found. If jobs of one type keep failing, it reads one payload before suggesting a retry, because if the payload is the problem they will all fail again. If a route has become slow, it reads the query plan and proposes an index, along with what `punch check` says the migration will cost. If the backup is behind, it cannot fix the storage provider's credentials, but it can tell you which file is behind and since when.

The agent proposes and you decide. It has its own user and its own token, so every request it makes carries its identity on its wide events, and you revoke the token when you are done. It cannot quietly close a problem either: a resolved error group reopens at the next occurrence.

None of this needs a monitoring vendor's API or a log pipeline. The data the agent reads is in a SQLite file next to the app, and the pages it reads are the same ones you would open.

Your on-call rotation is now you, an agent, and the cat, and only one of them is reading error groups at 3 a.m. Go outside. If something can't wait, Punch mails you; if it can, the agent will have a pull request ready when you get back.

## What Punch is not

Punch runs on one machine, with SQLite, on a local disk. It renders pages on the server and uses a few JavaScript modules without a build step, so it is not for a single-page app with a Node toolchain. Crystal's ecosystem is small, so if you need a deep library ecosystem or a large hiring pool, that is a real cost. Compile times matter too: Crystal has no incremental compilation, so a rebuild of a real app takes ten seconds or more, which is why `punch typecheck`, at a few seconds, is the check between edits.

Punch is at version 0.1 and not released. The API may change between minor versions, and it has not had an independent security review. The invoicing app it is tested with has organisations, roles, encrypted columns, public IDs (the IDs in URLs, which do not let anyone count records or guess the next one), and payment webhooks, and every code example in the documentation is checked against the code it quotes.

## Room to work

A small team does not get the room a large one has. There is no platform team and no budget for twelve services. So the framework has to do that work up front, in the compiler, the checks, and a running system that explains itself, so that one person and a few agents can carry a product from the first commit through every deploy after it.

Small team, full punch.
