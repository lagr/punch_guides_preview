# Coming from Rails

Applies to Punch 0.1.

Punch takes much of its direction from Rails 8: one machine, SQLite in production, jobs and caching without Redis, Hotwire instead of a JavaScript framework, Kamal for deploys. If you know Rails, you know what a Punch app is for. What changes is how the code is shaped, because Punch is built around a compiler: things Rails checks at run time, or by convention, Punch checks when the app compiles. This guide maps Rails habits to their Punch counterparts, and names the few things Punch deliberately lacks.

After reading this guide, you will know:

- where each part of a Rails app maps in a Punch app;
- how controllers, models, and views become operations, models, and components;
- why there are no validations, callbacks, or scaffolds, and what takes their place;
- which Ruby habits do not carry over, and what to do instead;
- which guides to read first.

Before you start: you have read the [Introduction](introduction.md). Read this beside [How a Punch App Works](how-a-punch-app-works.md).

## The map

| Rails | Punch | Where to read |
|---|---|---|
| `config/routes.rb` | Routes derived from operation names; `punch routes` lists them | [Operations and Routing](operations-and-routing.md) |
| Controller action | An operation: a `Punch::Query` or `Punch::Command`, one per file | [Operations and Routing](operations-and-routing.md) |
| Strong parameters | The operation's `Input`, parsed into domain types before `call` | [Input and Domain Types](input-and-domain-types.md) |
| `before_action :authenticate_user!` | The actor as a `call` parameter | [Authentication](authentication.md) |
| Pundit, CanCanCan | `@[Punch::Authorize(Policy)]`, required on every operation | [Authorization and Roles](authorization-and-roles.md) |
| Active Record model | Named SQL, generated records, and models in two tiers | [Domain Models](domain-models.md) |
| `validates` | Domain types, the input's `check`, unique indexes | [Domain Models](domain-models.md#where-each-kind-of-rule-goes) |
| Migrations in Ruby | Migrations in plain SQL, forward only; `db/schema.sql` | [Migrations](migrations.md) |
| `rescue_from RecordNotFound` | A `NotFound` result type with `@[Punch::Status(404)]` | [Results, Errors, and Status Codes](results-errors-and-status-codes.md) |
| ERB views, partials, helpers | Components: Crystal structs that write elements | [Pages and Components](pages-and-components.md) |
| `form_with model:` | `v.form(Command.action)`, bound to the Command's input | [Forms](forms.md) |
| `link_to … invoice_path(@invoice)` | `ShowPage.href(invoice_id: id)`, checked at compile time | [Pages and Components](pages-and-components.md#links-between-pages) |
| `config/locales/*.yml`, `t(".key")` | The same YAML; `t.invoices.title` as a generated method | [Translations and Formats](translations-and-formats.md) |
| Propshaft, importmap-rails | `Punch.assets`, an import map, Turbo built in | [Assets and JavaScript](assets-and-javascript.md) |
| Turbo Streams over Action Cable | Live pages over server-sent events | [Turbo Frames, Streams, and Live Pages](turbo-frames-streams-and-live-pages.md) |
| Jbuilder, serializers | Result structs; `openapi.json` derived and committed | [Building a JSON API](building-a-json-api.md) |
| Active Job with Solid Queue | Jobs in `queue.db`, enqueued through an outbox | [Background Jobs](background-jobs.md) |
| Callbacks, `ActiveSupport::Notifications` | Domain events and subscribers | [Domain Events](domain-events.md) |
| Action Mailer | Mail as a component, sent from a job | [Mail](mail.md) |
| Solid Cache | Typed caches in `cache.db`, forgotten by events | [Caching](caching.md) |
| `rails credentials:edit` | `punch creds:edit`, with a typed credentials struct checked at boot | [Credentials and Keys](credentials-and-keys.md) |
| `rack-attack` | `@[Punch::RateLimit]` on operations | [Rate Limits](rate-limits.md) |
| Kamal, Thruster | Kamal or systemd, Caddy or kamal-proxy in front | [Deploying](deploying.md) |
| RSpec request specs | Endpoint specs through `Punch::Spec.client`, checked against `openapi.json` | [Testing Punch Applications](testing-punch-applications.md) |
| `rails console` | No counterpart; `punch db:shell` for the database | below |
| Scaffolds | No counterpart; the reference slice and agents | below |

## Controllers become operations

A Rails controller groups actions by resource and shares filters between them. Punch has no controllers. Each action is an operation in a file of its own, and everything a filter would have done is declared on it:

```ruby illustrative="a Rails controller action, for comparison"
class InvoicesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_invoice

  def deliver
    authorize @invoice
    return render json: { error: "already sent" }, status: :conflict unless @invoice.draft?
    @invoice.update!(status: :sent, number: Invoice.next_number, sent_at: Time.current)
    InvoiceMailer.with(invoice: @invoice).invoice.deliver_later
    render json: @invoice
  end
end
```

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

- **The route** comes from the name: `Billing::Invoices::Send` is `POST /billing/invoices/:invoice_id/send`.
- **The signed-in user** is the `actor` parameter, and the policy is an annotation. An operation without either a policy or `@[Punch::Public]` does not compile, so there is no action that forgot its `authorize`.
- **The parameters** are the `Input`, parsed before `call` into an `ID`, never a string.
- **The outcomes** are types in `Output`, each with its status. The compiler makes `call` handle every state the store can return.
- **The mail** is not sent here. The Command publishes `InvoiceSent`, and a subscriber mails the invoice after the commit, with retries.

An operation is closer to a service object with a typed signature than to a controller action. It serves the JSON API, and pages call it too.

## Models: SQL first, behaviour on top

Active Record maps tables to classes and builds SQL from method calls. Punch goes the other way: you write SQL in `.sql` files, and `punch db:generate` turns each named query into a typed method, and each table into a record with its columns already in domain types:

```sql illustrative="a named query from reference/app/billing/invoices/queries.sql"
-- name: Organization :one
-- Which organization an invoice belongs to, for the policy.
SELECT organization_id FROM invoices WHERE id = :invoice_id;
```

`Billing::Invoices::Queries.organization(db, invoice_id: id)` then returns a row with `organization_id : Accounts::Organizations::ID`, or `nil`. There is no query builder, no lazy relation, and no N+1 by accident: each query is one statement you can read, and `punch check` fails one whose plan scans a whole table.

Behaviour comes in two tiers. Simple entities get methods on the generated record, as in a slim Active Record model. Entities with a lifecycle, such as invoices, get one type per state, so "only a draft can be sent" is the shape of the types rather than an `if` ([Domain Models](domain-models.md)).

**Models never touch the database.** There is no `invoice.save` and no `Invoice.find` inside a model. The operation, or a store it calls, loads and saves; the model decides.

## No validations

Rails puts every check in `validates` lines, run before saving. Punch puts each kind of rule where it can be enforced best:

```ruby illustrative="Rails validations, for comparison"
class Customer < ApplicationRecord
  validates :name, presence: true
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }, uniqueness: { scope: :organization_id }
end
```

| The rule | Punch |
|---|---|
| `name` must be present | `getter name : Punch::NonEmptyString`: a blank name never reaches `call` |
| `email` must be an address | `getter email : Punch::Email`: parsed, trimmed, lowercased at the boundary |
| `email` unique in the organization | A unique index; the generated insert returns `EmailTaken`, a 422 at `email` |

A value that reached a model is already valid, so a model checks only what depends on its own state. And uniqueness is decided by the database, not by a lookup first, which in Rails can race ([Input and Domain Types](input-and-domain-types.md), [Results, Errors, and Status Codes](results-errors-and-status-codes.md#field-errors-rejections-and-violations)).

## No callbacks

`after_create`, `after_commit`, and observers run code that the caller never sees. Punch has domain events instead: a Command publishes a fact (`InvoiceSent`), and subscribers react, after the commit by default, or as jobs when they call out of the app. `punch events` draws the graph of who publishes and who listens, so the reaction is never hidden ([Domain Events](domain-events.md)).

## Views are components

ERB templates are strings with holes; Punch's views are Crystal structs that write HTML element by element:

```erb illustrative="an ERB partial, for comparison"
<tr id="<%= dom_id(invoice) %>">
  <td><%= link_to invoice.number, invoice_path(invoice) %></td>
  <td><%= number_to_currency(invoice.total) %></td>
</tr>
```

```crystal illustrative="the same row as a component, in the reference app's style"
struct InvoiceRow
  include Punch::Component

  def initialize(@summary : Summary)
  end

  def view(v : Punch::View) : Nil
    v.tr(id: Punch.dom_id(@summary.id)) do
      v.td { v.a @summary.number.to_s, href: ShowPage.href(invoice_id: @summary.id) }
      v.td v.locale.money(@summary.total, @summary.currency)
    end
  end
end
```

Everything is escaped, links are typed, inline scripts and styles do not compile, and a renamed field fails the build instead of a page. A layout is a component that takes a block; a partial is a component another renders; a helper is a module function ([Pages and Components](pages-and-components.md)).

Forms work like `form_with` with Turbo's conventions built in: a successful Command answers 303, a refused one 422 with the page again, every error at its field. You never write the `if @customer.save … else render :new` branch ([Forms](forms.md)).

## Jobs, cache, and real-time without Redis

Like Rails 8's Solid Queue and Solid Cache, Punch keeps jobs and cache entries in SQLite files beside the database: `queue.db` and `cache.db`. Two differences matter:

- **Jobs from a Command go through an outbox** in the Command's own transaction, so a job exists exactly when the data it is about was committed. There is no `after_commit { SomeJob.perform_later }` to remember.
- **Job arguments are typed fields**, stored as JSON and checked across deploys: `punch check` fails a change that would make stored jobs unreadable ([Background Jobs](background-jobs.md)).

Real-time updates use server-sent events instead of WebSockets. A live page decides per viewer what each event changes, so there are no broadcast channels to secure ([Turbo Frames, Streams, and Live Pages](turbo-frames-streams-and-live-pages.md)).

## What Punch lacks on purpose

**No console.** There is no `rails console` to load the app and change data by hand. Change data through the app's own Commands, from a page or the API; inspect it with `punch db:shell`, which opens SQLite's shell with Punch's collations, and with `punch ops` for errors and jobs. Data that a console session would fix is a Command the app is missing.

**No admin.** The ops console at `/_punch` shows errors, jobs, mail, and performance to operators. It is not an admin for the app's data, and there is no generated one. Build the pages your users and staff need, with the same policies.

**No scaffolds.** Rails generators write boilerplate and teach conventions. Punch has less boilerplate, since parsers, routes, the API document, and form bindings are derived, and the conventions are taught by the reference app's billing slice, the app's `AGENTS.md`, and lint rules that reject drift. `punch generate` writes only what follows from the structure: outlines, an entity's types and column mapping ([Application Structure](application-structure.md#adding-a-namespace-or-an-entity)). Operations and pages you write, or an agent writes from the reference slice.

**No `method_missing`, no monkey-patching.** Crystal has neither as Ruby does, and Punch's DSLs are macros that run at compile time. What would be a dynamic finder in Ruby is a named query.

**No multiple databases or replicas.** One machine, SQLite, a writer and a pool of readers. That is the design, not a gap to fill later.

## Ruby habits that do not carry over

**`rescue` for control flow.** `find` raising `RecordNotFound` and a `rescue_from` turning it into a 404 is how Rails expresses a common outcome. In Punch, expected outcomes are values: `Store.load` returns `nil`, and `call` returns `NotFound`. An exception is a bug, and reaches the ops console as one.

**`nil` anywhere.** Crystal's `nil` is a type. A method that may return nothing says so in its signature (`Invoice::Any?`), and the compiler makes the caller handle it. `undefined method for nil` does not happen at run time.

**Mutable shared state.** Class variables, memoized globals, and a mutable `Current` work in a Rails process with one thread per request. Punch runs requests on several threads at once; anything a request needs comes in through `call`, and shared values are immutable.

**Strings for everything.** A Rails app passes IDs, emails, and money as strings and numbers. In Punch, each is its own type, and a function that takes a `Customers::ID` does not accept an `Invoices::ID`.

**Reaching for a gem.** Much of what Rails apps add as gems (authentication, authorization, rate limiting, idempotency keys, OpenAPI, backups, error tracking) is part of Punch, and built to work together. Before adding a shard, check the guides.

See [Crystal for Punch Developers](crystal-for-punch-developers.md) for the language itself.

## Where to start

Read in this order:

1. [How a Punch App Works](how-a-punch-app-works.md): the request from socket to response.
2. [Crystal for Punch Developers](crystal-for-punch-developers.md), unless you know Crystal.
3. [Operations and Routing](operations-and-routing.md), [Input and Domain Types](input-and-domain-types.md), and [Results, Errors, and Status Codes](results-errors-and-status-codes.md): the core.
4. [Domain Models](domain-models.md) and [Pages and Components](pages-and-components.md).

Then open the reference app's `app/billing/invoices/` beside the guides: it is the slice every convention is shown in.

## Common mistakes

**Looking for the routes file.** There is none. Run `punch routes`, and name the operation for the route you want.

**Writing a fat page.** A page that loads, checks, and saves is a controller action in disguise. Put the work in a Command and let the page render.

**Checking uniqueness before inserting.** The Rails reflex `exists?` then `create` races. Insert, and return the violation the generated insert gives you.

**Raising for a 404.** `raise NotFoundError` in an operation answers 500. Return `NotFound`.

**Adding a callback-like subscriber for everything.** Events are facts other parts react to, not a place to hide the main work of a Command. Do the work in the Command; publish the fact for the reactions.

## Limits

- This guide covers Rails only. Phoenix and Django readers will find most of the mapping applies: Ecto changesets and Django forms map to inputs and domain types, LiveView and Channels to live pages, Oban and Celery to jobs.

## Rules

- One operation per action, named for its route; no controllers. ([Controllers become operations](#controllers-become-operations))
- Rules go in types, the input's `check`, models, indexes, and Commands; never in a validation layer. ([No validations](#no-validations))
- Reactions are events and subscribers, never hidden callbacks. ([No callbacks](#no-callbacks))
- Expected outcomes are values; exceptions are bugs. ([Ruby habits that do not carry over](#ruby-habits-that-do-not-carry-over))
- Data changes through the app's Commands, never by hand in a console. ([What Punch lacks on purpose](#what-punch-lacks-on-purpose))

## Reference

- [Introduction](introduction.md): what Punch is, and who it is for.
- [Application structure](../structure.md): where each kind of file goes.
- [Glossary](glossary.md): the words Punch uses.
