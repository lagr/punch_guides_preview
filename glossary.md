# Glossary

The words Punch's guides and messages use, one definition each. Where a word has an everyday meaning too, the guides use it only in the sense given here. Each term links to the page that explains it.

## The app and its code

**App binary.** The program `crystal build` makes from an app: it serves requests and answers commands such as `bin/tally db:migrate` and `bin/tally routes`. Compare *tool*. ([Tooling](../tooling.md))

**Entity.** A namespace that owns tables: it maps them in its `columns.cr`, and everything about it lives in its directory, such as `app/billing/invoices/`. ([Application structure](../structure.md#slices-namespaces-entities))

**Generated file.** A file a `punch` command writes from a source, named `*.gen.cr` or `openapi.json`. It is committed and never edited by hand: change the source and run the command again. `punch check` fails when one is out of date. ([Queries](../queries.md#named-queries))

**Namespace.** A directory under `app/` and the Crystal module of the same name: `app/accounts/sign_up/` holds `Accounts::SignUp`. Directories of a kind, such as `operations/` and `pages/`, add no namespace. ([Application structure](../structure.md#slices-namespaces-entities))

**Outline.** The file beside a namespace directory, named like it (`app/billing/invoices.cr`): one documented module that says in plain words what the concept means and is for. ([Domain models](../models.md#the-outline-of-an-entity))

**Reference app.** The invoicing app in Punch's repository (`reference/`). The guides quote it as the finished example of every feature it uses.

**Slice.** A directory directly under `app/`, a bounded part of the domain such as `billing` or `accounts`. Another slice uses its data only through its operations. ([Application structure](../structure.md#slices-namespaces-entities))

**Tool.** The standalone `punch` program. It creates, generates, checks, and builds apps (`punch new`, `punch db:generate`, `punch check`) and passes commands it lacks to the app binary. Compare *app binary*. ([Tooling](../tooling.md))

## Operations

**Command.** An operation that changes state: a subclass of `Punch::Command`, whose `call` receives a `Punch::Writer` and runs in one transaction. "Command" with a capital letter always means this; a program you type in a shell is a *CLI command*. ([Operations and routing](../operations.md))

**Endpoint.** The HTTP side of an operation: its method and path. ([Operations and routing](../operations.md#routes-three-rules))

**Input.** An operation's `Input` struct, which includes `Punch::Input`. Parsing a request into it yields every field typed, or a rejection. ([Input](../input.md))

**Operation.** A Query, a Command, or a Page: business logic with a typed input and a typed result, free of HTTP. Its name decides its route. Never shortened to "ops". ([Operations and routing](../operations.md))

**Page.** An operation that answers people with HTML: a subclass of `Punch::Page` that reads, calls other operations, and returns a component. ([Views and pages](../views.md#pages))

**Query.** An operation that only reads: a subclass of `Punch::Query`, whose `call` receives a `Punch::Reader`. Capitalised and in code when it means the type; SQL in a `.sql` file is a *named query*. ([Operations and routing](../operations.md))

**Surface.** One of the three forms an app answers in: HTML for people, markdown for agents, JSON for programs. ([Views and pages](../views.md#pages-agents-read))

## Results and failures

**Invalid.** One value that failed to parse, as a `Punch::Invalid`: a code and its parameters, never a sentence. ([Core types](../core-types.md#invalid-and-rejection))

**Problem.** A failure result an endpoint answers with, rendered as Problem Details (RFC 9457) with a snake_case `code`. A problem type includes `Punch::Problem`. ([JSON endpoints and OpenAPI](../openapi.md#problem-details))

**Rejection.** A whole input that failed to parse, as a `Punch::Rejection`: every error, each at its path. It answers 422. ([Core types](../core-types.md#invalid-and-rejection))

**Result.** What `call` returns: one type, or a union of the outcomes it can have (`Sent | AlreadySent | NoLines`), each with its status. Expected failures are results, never exceptions. "Error" in the guides means a bug or an exception. ([Core types](../core-types.md#results-are-unions))

**Violation.** A write the database refused because a unique index already holds the value, returned as a typed struct such as `EmailTaken` instead of raised. ([Queries](../queries.md#violations))

## Types

**Domain primitive.** A type for one kind of value, such as `Punch::Email` or `Punch::Money`, that can only be made by parsing: once you hold one, it is valid. ([Core types](../core-types.md#domain-primitives))

**Secret.** A value wrapped in `Punch::Secret(T)`, which prints as `[REDACTED]` and cannot be serialized or used as a hash key; `unwrap` it at the point of use. ([Core types](../core-types.md#secrett))

**Typed ID.** A record's ID as its own type, a subclass of `Punch::TypedID`, so an invoice's ID cannot be passed where a customer's is expected. ([Core types](../core-types.md#typed-ids))

## Data

**Declared clause.** A line in a named query's header that adds an optional filter (`-- when status:`) or a sort order (`-- order issued:`), chosen at the call by a typed `Filter` and an order enum instead of SQL assembled in Crystal. ([Queries](../queries.md#filters-and-sorting-declared-clauses))

**Keyset page.** One page of a list, read after the last row of the previous page (its cursor) rather than at an offset. A `:page` query returns `Punch::Paged`. ([Queries](../queries.md#pages-keyset-pagination))

**Migration.** A plain SQL file in `db/migrations/` that changes the schema, applied once and never edited afterwards. ([Migrations](../migrations.md))

**Named query.** SQL in a `.sql` file under a `-- name:` header, which `punch db:generate` turns into a typed method. Punch has no ORM and no query builder. ([Queries](../queries.md#named-queries))

**Record.** The struct `punch db:generate` writes for a row of a mapped table, reused by every query that selects the whole row. ([Queries](../queries.md#records-and-per-table-crud))

**Row.** The struct generated for one result row of a named query that selects something other than a whole record.

**State type.** One hand-written type per state of an entity's lifecycle, such as `Invoice::Draft` and `Invoice::Sent`, so only a draft has the methods a draft allows. ([Domain models](../models.md#tier-2-state-types-and-a-store))

**Store.** The module that loads an entity's rows into its state types and saves them back. ([Domain models](../models.md#tier-2-state-types-and-a-store))

**Storage.** The directory named by `PUNCH_STORAGE` that holds an app's four database files: `primary.db`, `queue.db`, `ops.db`, and `cache.db`. ([Persistence](../persistence.md#databases-and-connections))

## Access

**Actor.** Whoever a request acts for, as the app's own type: a signed-in user, a token's owner, or nobody. ([Authentication](../auth.md#the-actor))

**Grant.** A `Punch::Grant(O)`: proof that operation *O*'s policy allowed the call. A Command with a policy cannot run without one. ([Authorization](../authorization.md#grants))

**Policy.** The rule that decides who may run an operation. Every operation declares one, or says it is public. ([Authorization](../authorization.md#policies))

**Role.** A named set of permissions held by a user, in one organization or for the whole app. ([Authorization](../authorization.md#roles))

## Configuration

**Credentials.** What the app declares it needs from its deployment, API keys and its own settings alike, as a type that includes `Punch::Credentials`. Values come from the environment or an encrypted file. ([Credentials](../credentials.md))

**Key ring.** The app's root secret and the keys derived from it, for values that must be signed or encrypted. ([Key ring](../keys.md))

**Settings.** The `PUNCH_*` environment variables that configure Punch itself, such as `PUNCH_STORAGE` and `PUNCH_ENV`. Compare *credentials*. ([Tooling](../tooling.md))

## Background work

**Event.** A typed fact a Command publishes in its transaction (`InvoiceSent`), named in the past tense. Subscribers react to it after the commit. ([Domain events](../events.md#events))

**Idempotency key.** A client's name for one attempt at a Command, sent as the `Idempotency-Key` header, so a retry gets the stored answer instead of running twice. ([Operations and routing](../operations.md#idempotency-keys))

**Job.** A typed struct stored in `queue.db` and run by a worker outside the request, retried when it fails. ([Background jobs](../jobs.md#jobs))

**Outbox.** The table a Command writes its events and jobs to in its own transaction, so they are delivered only if it commits, and always then. ([Background jobs](../jobs.md#from-a-command-the-outbox))

**Payload lock.** The committed file `payloads.lock.json` with the shape of every job payload, checked so a deploy can still read the jobs already queued. ([Background jobs](../jobs.md#payloads-across-deploys))

**Subscriber.** Code that reacts to an event, run after the publishing Command has committed. ([Domain events](../events.md#delivery))

## Running

**Ops.** What operators need: the console at `/_punch`, the `punch ops` commands, and `ops.db`, where errors, job runs, and slow requests are recorded. ([Ops data](../ops.md))

**Wide event.** The one record per request (or job) that carries everything about it: route, outcome, status, database totals, and any field the code adds. It is also the root span of the request's trace. ([Tracing](../tracing.md#what-a-request-records))
