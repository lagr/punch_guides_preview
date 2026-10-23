# Operations and Routing

Applies to Punch 0.1.

The business logic of a Punch app lives in operations: one struct per thing the app can do, such as showing an invoice, creating a customer, or sending an invoice. An operation knows nothing about HTTP. It declares its input, says what it returns, and does its work in `call`. Punch derives everything else from its name and its types: the route, the HTTP method, the OpenAPI entry, the permission it needs. You never write a routes file.

After reading this guide, you will know:

- when an operation is a Query and when it is a Command, and what each may do;
- how an operation's name and input decide its route, and how to check the result;
- how to build paths without concatenating strings;
- what an operation can ask for in `call`, and why it asks instead of reaching for globals;
- how a browser and a program share one path;
- what the compiler says when two routes collide or a reserved name is misused.

Before you start: you know how an operation's input is declared ([Input and Domain Types](input-and-domain-types.md)). The examples come from the reference app's invoices, in `reference/app/billing/invoices/operations/`.

## Queries read, Commands write

Every operation is one of two kinds, and the kind is a type:

| Kind | `call` receives | Answers | Runs in |
|---|---|---|---|
| `Punch::Query` | a `Punch::Reader`, which cannot write | `GET` | a read transaction |
| `Punch::Command` | a `Punch::Writer` | `POST`, `PATCH`, `DELETE` | one write transaction |

The reference app's Query that shows one invoice:

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

A Query cannot write: a `Punch::Reader` has no method that changes data, so a Query that tries does not compile. A Command runs inside one transaction. If it returns a failure, a result with a status from 400 on, everything it wrote is rolled back. If it raises, the same happens and the client gets a 500. There is no half-done Command.

The split is what lets Punch answer `GET` safely from any number of threads, retry a Command after a lost answer ([Idempotency keys](../operations.md#idempotency-keys)), and tell an agent which actions change something.

> [!NOTE]
> **Coming from Rails:** there are no controllers. An operation is closer to a service object with a typed signature, and the route comes from its name rather than from `config/routes.rb`. A page that shows HTML is a third kind of operation, `Punch::Page`, which reads like a Query and renders components ([Views and pages](../views.md#pages)).

## One file per operation

Each operation has its own file in the `operations/` directory of its namespace, named after it:

```text
app/billing/invoices/operations/
  index.cr     Billing::Invoices::Index      GET    /billing/invoices
  show.cr      Billing::Invoices::Show       GET    /billing/invoices/:invoice_id
  create.cr    Billing::Invoices::Create     POST   /billing/invoices
  update.cr    Billing::Invoices::Update     PATCH  /billing/invoices/:invoice_id
  destroy.cr   Billing::Invoices::Destroy    DELETE /billing/invoices/:invoice_id
  send.cr      Billing::Invoices::Send       POST   /billing/invoices/:invoice_id/send
```

The file holds the operation's annotations, its `Input`, its `call`, and the result types only it returns. Open one file and you see everything the operation can answer. Results several operations share go in the namespace's `results.cr`. Listing `operations/` lists the namespace's API. `punch check` reports an operation anywhere else under `app/`, with the path it belongs at ([Application structure](../structure.md)).

## How a name becomes a route

Three rules turn an operation's full name and its input into a route:

1. **The namespace is the path.** `Billing::Invoices` becomes `/billing/invoices`. Names are underscored, so `LineItems` becomes `line_items`.
2. **Five names have REST shapes.** `Index` and `Show` are `GET` and must be Queries. `Create` is `POST`, `Update` is `PATCH`, and `Destroy` is `DELETE`, and those must be Commands. Any other name becomes a last segment: `GET` for a Query, `POST` for a Command. So `Send` is `POST …/send`.
3. **Path parameters become segments.** A path parameter named after a namespace segment, `invoice_id` after `invoices`, goes right after that segment. Other path parameters are appended in the order they are declared.

`Send` declares one path parameter, `invoice_id`, so it is `POST /billing/invoices/:invoice_id/send`:

```crystal source=reference/app/billing/invoices/operations/send.cr select=Billing::Invoices::Send elide=call
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
      # ...
    end
  end
end
```

Rule 3 is what nests routes. Members belong to an organization, so `Accounts::Organizations::Members::Destroy` takes `organization_id` and `user_id`:

```text
DELETE  /accounts/organizations/:organization_id/members/:user_id
```

`organization_id` follows `organizations`, and `user_id`, which follows no segment, comes last.

When two routes could match a request, a literal segment wins over a parameter. That is why `GET /billing/invoices/new`, a page, and `GET /billing/invoices/:invoice_id` live side by side. A known path with an unknown method answers 405 with an `Allow` header, and `HEAD` is answered by the `GET` route.

## Checking the routes

`punch routes` builds the app and prints what Punch derived. In the reference app, part of it reads:

```console
$ punch routes
GET     /billing/invoices                                          json                Billing::Invoices::Index
GET     /billing/invoices                                          html,markdown,live  Billing::Invoices::IndexPage
POST    /billing/invoices                                          json,html,markdown  Billing::Invoices::Create  (form on Billing::Invoices::CreatePage)
DELETE  /billing/invoices/:invoice_id                              json,html,markdown  Billing::Invoices::Destroy  (form on Billing::Invoices::ShowPage)
GET     /billing/invoices/:invoice_id                              json                Billing::Invoices::Show
GET     /billing/invoices/:invoice_id                              html,markdown,live  Billing::Invoices::ShowPage
PATCH   /billing/invoices/:invoice_id                              json                Billing::Invoices::Update
GET     /billing/invoices/:invoice_id/link                         json                Billing::Invoices::Link
POST    /billing/invoices/:invoice_id/send                         json,html,markdown  Billing::Invoices::Send  (form on Billing::Invoices::ShowPage)
GET     /billing/invoices/new                                      html                Billing::Invoices::CreatePage
...
```

Each line is a method, a path, the surfaces it answers (JSON for programs, HTML for browsers, markdown for agents, `live` for pages that update in the browser), and the operation. `punch routes --json` gives the same as data, with each route's operation ID, policy, and rate limits, for scripts and agents.

Check the table when you add or rename an operation. A route is part of the app's contract: renaming `Send` to `Deliver` changes its path for every client.

## When a route is wrong

Two operations that answer the same method and path do not compile, and the error names both:

```crystal fails=spec/fail/route_collision.cr
module Invoices
  @[Punch::Public]
  @[Punch::RateLimitExempt("a fixture")]
  struct Archive < Punch::Command
    struct Input
      include Punch::Input
      @[Punch::PathParam]
      getter invoice_id : Int64
    end

    def call(input : Input, db : Punch::Writer) : Nil
    end
  end
end

@[Punch::Route("POST", "/invoices/:id/archive")]
@[Punch::Public]
@[Punch::RateLimitExempt("a fixture")]
struct ArchiveAlias < Punch::Command
  struct Input
    include Punch::Input
    @[Punch::PathParam]
    getter id : Int64
  end

  def call(input : Input, db : Punch::Writer) : Nil
  end
end

Punch::Routing.routes
```

```text
Error: Route collision: ArchiveAlias and Invoices::Archive both answer POST /invoices/:id/archive. Rename one, or give one @[Punch::Route].
```

A reserved name on the wrong kind does not compile either:

```crystal fails=spec/fail/route_command_as_index.cr
module Invoices
  @[Punch::Public]
  @[Punch::RateLimitExempt("a fixture")]
  struct Index < Punch::Command
    struct Input
      include Punch::Input
    end

    def call(input : Input, db : Punch::Writer) : Nil
    end
  end
end

Punch::Routing.routes
```

```text
Error: Invoices::Index is a Command, but `Index` is reserved for Queries (GET).
```

The fix is almost always a better name. A Command that lists something is not an `Index`; name it after what it does.

## Overriding a route

`@[Punch::Route(method, path)]` replaces all three rules. It is for the rare route the rules cannot express. The home page that `punch new` writes is one, since no name derives `/`:

```crystal source=reference/app/home/pages/index_page.cr select=Home::IndexPage
# app/home/pages/index_page.cr
module Home
  # ...

  # `GET /`: the first page after signing in (`Accounts::Sessions::CreatePage`
  # and the other sign-in pages redirect here). A browser without a session
  # goes to sign in.
  @[Punch::Authorize(Punch::SignedIn)]
  @[Punch::Route("GET", "/")]
  struct IndexPage < Punch::Page
    struct Input
      include Punch::Input
    end

    alias Output = Welcome

    def call(input : Input, db : Punch::Reader, actor : Invoicing::Actor) : Output
      Welcome.new(actor)
    end
  end
end
```

The path's `:name` segments must be exactly the input's path parameters, and a Command cannot take `GET`. Prefer a better name to an override: overridden routes are where a reader has to look twice.

## Building paths

Every operation has a typed path helper, generated from its route:

```crystal illustrative="the helper on its own; the reference app's specs and pages use it throughout"
Billing::Invoices::Send.path(invoice_id: invoice.id)   # => "/billing/invoices/42/send"
```

A missing, unknown, or wrongly typed parameter does not compile:

```crystal fails=spec/fail/path_missing_param.cr
module Invoices
  @[Punch::Public]
  struct Show < Punch::Query
    struct Input
      include Punch::Input
      @[Punch::PathParam]
      getter invoice_id : Int64
    end

    def call(input : Input, db : Punch::Reader) : Nil
    end
  end
end

Invoices::Show.path
```

```text
Error: Invoices::Show.path: missing path parameter `invoice_id` (Int64)
```

Values are escaped, and enums appear as their snake_case names, as the input parser reads them. So never build a path by joining strings: a renamed operation or parameter then breaks at compile time instead of in production. Pages have `href` for the same purpose, and a Command has `action` for forms ([Views and pages](../views.md#links)).

## What `call` can ask for

`call` takes the input and the database handle first. After them it may ask for more, in any order; Punch recognizes each by its type and passes it in:

| Parameter type | Gives | In |
|---|---|---|
| the app's actor, such as `Invoicing::Actor` | who the request acts for; nilable when signing in is optional | all |
| `Punch::Grant(Op)` | proof that the operation's policy allowed the call | operations with a policy |
| `Punch::Clock` | the time, as `clock.now` | all |
| `Punch::Locale` | the request's language and formats | all |
| `Punch::Events(A \| B)` | publishing events, in the transaction | Commands |
| `Punch::Jobs(A \| B)` | enqueuing jobs, in the transaction | Commands |
| `Punch::RawRequest` | the body as received, for signatures | Commands |

There are also the parameters for signing in and checking passwords ([Authentication](../auth.md)). `Send` above asks for five: its actor, its grant, `Punch::Events(InvoiceSent)`, `Punch::Jobs(Remind)`, and the clock.

Asking in `call`, instead of calling `Time.utc` or a global queue, makes an operation's dependencies part of its signature. A spec can set the clock. The compiler can list every event an operation publishes and every job it enqueues, which is how `punch events` draws the app's event graph. And a Query that asks for `Punch::Events` does not compile, because only a Command may change something.

A parameter of a type Punch does not know is underlined in `call`, in the operation's own file, not somewhere in the router.

## Who may run it

Every operation says who may call it: `@[Punch::Authorize(Policy)]` with a policy, or `@[Punch::Public]`. A Command that runs without a signed-in user also needs a rate limit, `@[Punch::RateLimit(…)]`, or `@[Punch::RateLimitExempt("reason")]`. An operation without them does not compile. `Send` above shows all three of its annotations: a policy, a rate limit, and a required idempotency key. Policies are the subject of [Authorization](../authorization.md), rate limits of [Rate limits](../rate-limits.md).

## A browser and a program on one path

A Query and a Page can share a path. `Billing::Invoices::Show` and `Billing::Invoices::ShowPage` both answer `GET /billing/invoices/:invoice_id`. The router picks by the request's `Accept` header: a client that prefers `text/html`, as every browser does, gets the page; anything else, `curl` and `fetch` included, gets the JSON. Both answers carry `Vary: Accept`.

So one URL works for a person in a browser, a program, and an agent asking for markdown. Page names follow the same rules, without the `Page` suffix, always with `GET`, plus `CreatePage` at `…/new` and `UpdatePage` at `…/:id/edit` for the pages that hold a Command's form ([Views and pages](../views.md#routes)).

## From request to response

For each request, the router:

1. matches the route and takes the path parameters from it;
2. parses the input: from the query string for a Query or a Page, from the body for a Command (JSON, or a form from a page), at most 1 MiB unless the operation raises the limit with `@[Punch::BodyLimit(bytes)]`, and answers 422 with every error if it does not parse;
3. applies the rate limits by client address and input, and answers 401 when the operation needs a signed-in user and there is none;
4. opens the transaction, a read one or a write one, loads the actor, and runs the policy inside it, so a concurrent change cannot race the decision;
5. runs `call`;
6. renders the result with its status ([Results, Errors, and Status Codes](results-errors-and-status-codes.md)), and rolls a Command's transaction back if the result is a failure.

## Common mistakes

**A Command named `Index` or `Show`.** The compile error above. Name it after its action: `Recalculate`, not `Index`.

**Building a path by hand.** `"/billing/invoices/#{id}/send"` keeps working after `Send` is renamed and then answers 404. Use `Send.path(invoice_id: id)`.

**Overriding a route to get a nicer URL.** `@[Punch::Route]` hides the route from the rules every reader knows. Rename the operation or move it to the namespace its URL suggests.

**Reaching for globals in `call`.** `Time.utc`, `Punch::Job::Queue.enqueue`, or a class variable makes the operation untestable and hides what it does. Ask for `Punch::Clock`, `Punch::Jobs(…)`, or `Punch::Events(…)` in `call`.

**Writing in a Query.** A Query that needs to record something, such as "last viewed", is a Command, or it hands the write to a job. A `Reader` has no way to write.

**Calling another system from a Command.** The call happens inside the write transaction and cannot be rolled back. Enqueue a job instead ([Background Jobs](background-jobs.md)).

## Limits

- Routes are derived at compile time; there are no routes added at run time.
- One operation answers one method and path. Two paths for one action are two operations.
- An operation cannot answer two methods.

## Rules

- A Query reads; a Command writes, in one transaction. Choose the kind by whether the operation changes state. ([Queries read, Commands write](#queries-read-commands-write))
- One file per operation, in its namespace's `operations/` directory, named after it. ([One file per operation](#one-file-per-operation))
- Let the name decide the route; check it with `punch routes`. Override with `@[Punch::Route]` only when no name expresses it. ([How a name becomes a route](#how-a-name-becomes-a-route), [Overriding a route](#overriding-a-route))
- Build paths with `Operation.path(...)`, never by concatenating strings. ([Building paths](#building-paths))
- Ask for the clock, events, jobs, and the actor in `call`; never use `Time.utc` or a global queue in an operation. ([What `call` can ask for](#what-call-can-ask-for))
- Every operation has a policy or `@[Punch::Public]`. ([Who may run it](#who-may-run-it))

## Reference

- [Operations and routing](../operations.md): matching, every `call` parameter, idempotency keys, and the raw request.
- [Views and pages](../views.md#routes): page routes and how they share paths with Queries.
- [JSON endpoints and OpenAPI](../openapi.md): how results become responses and the OpenAPI document.
- [Application structure](../structure.md): where every kind of file goes.
