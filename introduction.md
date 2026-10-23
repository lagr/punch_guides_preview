# Introduction: What Punch Is

Applies to Punch 0.1.

Punch is a web framework for Crystal in which the compiler carries the load. Requests are parsed into types where they arrive, business logic is typed operations, the API's OpenAPI document is derived from those types, and jobs, backups, secrets, and telemetry come with the framework instead of being assembled from services. An app deploys as one binary and a few SQLite files on one machine. In short: Rails 8's "no PaaS, SQLite, solid everything" direction, with a compiler.

After reading this guide, you will know:

- whether Punch fits your project, and the limits that rule it out;
- the principles you will meet in every guide;
- what a Punch app is, on disk and in production;
- which guides to read next.

Before you start: nothing.

## Three surfaces from one set of operations

A Punch app is a set of operations: a Query that shows an invoice, a Command that sends one. Each is a struct with a typed input and a typed set of outcomes:

```crystal source=reference/app/billing/invoices/operations/show.cr
# app/billing/invoices/operations/show.cr
module Billing::Invoices
  @[Punch::Authorize(ByRole)]
  struct Show < Punch::Query
    struct Input
      include Punch::Input

      @[Punch::PathParam]
      getter invoice_id : ID
    end

    alias Output = Detail | NotFound

    def call(input : Input, db : Punch::Reader, _actor : Invoicing::Actor, clock : Punch::Clock) : Output
      invoice = Store.load(db, input.invoice_id)
      invoice ? Detail.new(invoice, clock.now) : NotFound.new
    end
  end
end
```

From that one declaration, Punch serves three surfaces:

- **HTML for people.** A page beside the Query renders the invoice, with the buttons the viewer may use.
- **Markdown for agents.** The same page, asked for as markdown, describes the invoice and its actions, so a coding or operations agent can read and act on the app as a person would.
- **JSON for programs.** The Query answers JSON, documented in a committed `openapi.json` that is derived from the types, never written by hand.

The route, the input parser, the form fields, the error messages, and the OpenAPI schema all come from the same types, so they cannot drift apart.

## The principles

You will meet these in every guide. Each comes with the constraint it puts on an app:

- **Illegal states don't compile.** If an operation can end three ways, its type says so, and every caller handles all three. A missing branch, a missing policy, a failure without a status: compile errors, not production bugs.
- **Parse at the boundary, trust inside.** Every value that enters, from a request, a database row, a job payload, or a credential, is parsed once into a type that can only hold valid values. Inside an operation, there is no `String` that should have been an email address.
- **Expected failures are values.** "Not found" or "already sent" is a result the operation returns. Exceptions are for bugs.
- **One declaration, many projections.** Routes, parsers, forms, documentation, and links are derived from the types. If two could disagree, the design is wrong.
- **Generated files are committed.** Typed queries, translations, the OpenAPI document, and the payload lock are files in the repository that a reviewer can read, and `punch check` fails when one is stale.
- **SQL, HTML, and OpenAPI, not new languages.** Queries are SQL files, pages are HTML elements, the API contract is OpenAPI. People and coding agents already know them.
- **Side effects are announced.** A Command publishes typed events; reactions live in subscribers the compiler knows about.
- **Observable by default.** One wide event per request and per job, an ops console in the app, no third-party service needed.
- **Secure by default.** Escaping, secrets, cross-site checks, and SQL parameters are types and checks, not habits.

## One binary, one machine, a few files

In production, a Punch app is:

- **One static binary,** with its code, migrations, assets, encrypted credentials, and OpenAPI document inside. It runs on any Linux of its architecture without libraries.
- **A storage directory** on the machine's local disk, holding four SQLite files: the app's data, the job queue, the ops data, and a cache.
- **Litestream beside it,** replicating the data to S3-compatible storage within a second.
- **A reverse proxy** in front, which terminates TLS: kamal-proxy or Caddy.

There is no Redis, no separate job service, no Node toolchain, and no database server. A deploy starts the new binary beside the old one and hands over without dropping requests ([Deploying](deploying.md)).

On disk, an app made by `punch new` holds its code under `app/`, one directory per part of the domain, its SQL beside the code that uses it, its migrations in `db/migrations/`, and its texts in `config/locales/`. A type's name says where its file is ([Application structure](../structure.md)).

## Who Punch is for

Punch is for small teams, from one developer to about five, building business software: SaaS for other businesses, internal tools, or content with an API. It suits developers coming from Rails, Phoenix, or Hanami who want types to catch what tests miss, who run on one or two servers, or who ship software their customers host themselves. It is built for teams that write much of their code with coding agents and review it themselves, so its conventions are written for people and agents alike.

It is a good fit for an API-first backend whose clients are generated from the OpenAPI document, and for self-hosted products, where one binary and a few files are an unusually simple thing to hand a customer.

## What Punch is not for

- **More than one machine.** Punch runs on one node. There are no multi-writer databases, no read replicas, no automatic failover.
- **Databases other than SQLite.** SQLite is the only engine, used for what it is good at.
- **Platforms without a persistent local disk.** Stateless container platforms and network filesystems cannot hold SQLite's files safely.
- **Single-page apps with a Node toolchain.** Pages are server-rendered HTML with Turbo; JavaScript is a few modules without a build step.
- **Organizations that need a large hiring pool** and a deep library ecosystem: Crystal's is small.

Punch is at version 0.1. It is not released, its API may change between minor versions, and it has not yet had an independent security review.

## How to read these guides

The guides are grouped from a first app to production ([the index](README.md)). Each starts with what you will know after reading it and ends with its rules, the invariants in short. Code is quoted from two apps in Punch's repository: the reference app, a full invoicing app in `reference/`, and the guides' examples app, for what the reference app does not use. Every quote is checked against its source.

Each guide links to a reference page, one per subsystem in `lib/punch/docs/`, which lists every option. The [glossary](glossary.md) defines the words Punch uses.

Where to start:

- **New to Crystal:** [Crystal for Punch Developers](crystal-for-punch-developers.md), then [Operations and Routing](operations-and-routing.md).
- **Coming from Rails:** [Operations and Routing](operations-and-routing.md), [Input and Domain Types](input-and-domain-types.md), and [Results, Errors, and Status Codes](results-errors-and-status-codes.md), which say where your habits change.
- **Running an app:** [Deploying](deploying.md), [Backup and Restore](backup-and-restore.md), and [The Ops Console](the-ops-console.md).
- **A coding agent:** the app's `AGENTS.md` first, then [When the Compiler Says No](when-the-compiler-says-no.md) for any message you meet.

## Reference

- [The guides](README.md), by group.
- [The docs index](../llms.txt), every guide and reference page in one list.
