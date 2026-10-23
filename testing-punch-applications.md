# Testing Punch Applications

Applies to Punch 0.1.

Punch apps are tested with Crystal's own `Spec`, plus helpers from `punch/spec`. Every example gets a fresh database, requests go through the real router in process, every JSON response is checked against the committed `openapi.json`, and clocks, events, jobs, and mail can all be observed and controlled. Most of an app's specs are therefore short endpoint specs that exercise the real stack, with no mocks.

After reading this guide, you will know:

- how specs get a fresh database, and why that needs no cleanup;
- how to test an operation through a request, and what the contract check gives you for free;
- how to test pages, forms, and agents' pages;
- how to test events, jobs, mail, and webhooks end to end;
- how to control time in specs;
- what to test at which level, and what to leave untested.

Before you start: you know [How a Punch App Works](how-a-punch-app-works.md). The examples come from the reference app's `spec/` directory.

## Running specs

Specs mirror `app/`: `spec/billing/invoices/send_spec.cr` tests `app/billing/invoices/operations/send.cr`. Journeys across namespaces are in `spec/journeys/`, and factories per slice in `spec/support/`. The runner is `crystal spec`, with its usual tags, focus, and random order:

```console
$ crystal spec spec/billing/invoices/
$ crystal spec spec/billing/invoices/send_spec.cr:33
```

`punch check` builds all specs into one binary and runs them on several threads (`CRYSTAL_WORKERS`, 4 by default), then runs them again with the cache turned off. Running on several threads means the code under test runs as it does in production, so a shared mutable global shows up in specs, not on the first busy day.

The spec helper requires `punch/spec` and the app's library, and sets the credentials specs use:

```crystal source=reference/spec/spec_helper.cr
# spec/spec_helper.cr
require "punch/spec"
require "../src/invoicing"

# What every spec file shares; the factories are per slice.
module InvoicingSpec
  def self.parse(type : T.class, raw) : T forall T
    type.parse(raw).as(T)
  end
end

require "./support/accounts"
require "./support/billing"

# A key for specs only; real keys come from `punch keys:rotate`.
Punch::Spec.credentials(Invoicing::Credentials, <<-YAML)
  secret_key_base: 1:2026-10-09:#{"5a" * 32}
  issuer:
    name: Example Consulting GmbH
    vat_id: DE123456789
  payments:
    webhook_secret: #{BillingSpec::WEBHOOK_SECRET}
  mail:
    delivery: capture
    from: invoices@example.org
    from_name: Example Consulting
  YAML
```

## A fresh database per example

`Punch::Spec.db` is the current example's database: in memory, with `db/schema.sql` loaded on first use, and closed after the example. Opening one takes about half a millisecond, so every example starts from an empty schema without transactions to roll back or tables to truncate. The code under test can commit, fail, and open its own transactions, as it does in production.

A factory writes what an example needs directly, through the app's generated tables:

```crystal source=reference/spec/support/billing.cr select=BillingSpec.customer
# spec/support/billing.cr
module BillingSpec
  # ...

  # A customer of *organization*, by default of a new user's.
  def self.customer(db : Punch::Writer, email = "ada@example.org", organization : Accounts::Organizations::ID? = nil) : Billing::Customers::Record
    organization ||= AccountsSpec.organization(db, AccountsSpec.user(db).id)
    Punch.cannot_violate(Billing::Customers::Table.insert(db, organization_id: organization, name: InvoicingSpec.parse(Punch::NonEmptyString, "Ada Lovelace"), email: InvoicingSpec.parse(Punch::Email, email), created_at: Time.utc))
  end
end
```

Keep factories per slice in `spec/support/<slice>.cr`, and give them only the arguments specs vary. Prefer creating data through the app's own Commands in endpoint specs; reach for a factory when the Command is not what the spec is about.

## Endpoint specs

Most specs send requests. `Punch::Spec.client` runs the app's router in process, over this example's database: the policy, the transaction, the handles `call` asks for, and the rendering all run as in production, without a socket.

```crystal source=reference/spec/billing/invoices/create_spec.cr select="invoices API refuses an unknown customer and collects every input error with its path"
# spec/billing/invoices/create_spec.cr
# ...

describe "invoices API" do
  # ...

  it "refuses an unknown customer and collects every input error with its path" do
    client = AccountsSpec.client
    unknown = client.post(Billing::Invoices::Create.path, json: {customer_id: 99, currency: "EUR"})
    {unknown.status, unknown.json["code"]}.should eq({HTTP::Status::UNPROCESSABLE_ENTITY, "unknown_customer"})

    response = client.post(Billing::Invoices::Create.path, json: {
      customer_id: 1, currency: "eur",
      lines: [{description: "Fine", quantity: 1, unit_price: 100}, {description: " ", quantity: 0, unit_price: 1}],
    })
    response.json["errors"].as_a.map { |error| {error["path"].as_a.map(&.raw), error["code"].as_s} }.should eq([
      {["currency"], "invalid_currency"},
      {["lines", 1, "description"], "blank"},
      {["lines", 1, "quantity"], "out_of_range"},
    ])
  end

  # ...
end
```

- **Build paths with `Operation.path(...)`**, as the app does, so a renamed route breaks the spec at compile time.
- **`response.json`, `status`, `headers`, and `trace`** are there to check; `trace` holds every span of the request, so a spec can check which queries ran (`should have_span("Billing::Invoices::Queries.save")`).
- **An exception in an operation fails the example with it.** The client raises `Punch::Spec::ServerError`, with the request line and the original exception and backtrace, instead of returning a bare 500. A spec that tests the 500 itself uses `Punch::Spec.client(raise_errors: false)`.
- **A signed-in client** comes from the accounts slice's support file (`AccountsSpec.client` in the reference app), which signs up a user with an organization.

## The contract check

Every JSON response an endpoint spec receives is checked against the app's committed `openapi.json`: the status must be documented, the content type named, and the body must match the schema. A response that breaks the document fails the example at the line of the request:

```text
GET /billing/invoices/7 answered 200, which breaks the OpenAPI document:
  body (root): "due_on" is missing (required)
If the code is right, the document is stale: run 'punch openapi'.
```

So every endpoint spec is also a contract test, without any extra code. It cannot be turned off. When it fails, either the code broke the contract, or the document is stale and `punch openapi` brings it up to date. Requests are not checked, since specs send invalid input on purpose. See [Building a JSON API](building-a-json-api.md#why-the-document-is-committed).

## Calling an operation directly

An operation is a struct with a `call` method, so a spec can call it without a request, building what `call` takes with the helpers: `Punch::Spec.input` for the input, `Punch::Spec.grant` for the policy's grant, `Punch::Spec.events` and `Punch::Spec.jobs` for the handles, `Punch::Clock.at(time)` for the clock.

```crystal illustrative="the shape of a direct call; the reference app's specs go through the client"
input = Punch::Spec.input(Billing::Invoices::Send::Input, {"invoice_id" => id.to_s})
result = Punch::Spec.db.write do |db|
  send = Billing::Invoices::Send
  grant = Punch::Spec.grant(send, actor, db, invoice_id: id)
  send.new.call(input, db, actor, grant, Punch::Spec.events(send, db), Punch::Spec.jobs(send, db), Punch::Clock.at(Time.utc))
end
result.should be_a(Billing::Invoices::Detail)
```

That is more code than a request, and it skips the contract check, so the reference app does not do it. Reserve it for logic that is hard to reach through a request; most such logic belongs in a model, which is easier to test still.

## Models without a database

Models and generated rows are plain structs, so their specs need no database and no request:

```crystal source=reference/spec/billing/invoices/invoice_spec.cr select="Billing::Invoices::Invoice sends a draft: numbered, sent now, due after the payment term"
# spec/billing/invoices/invoice_spec.cr
# ...

describe Billing::Invoices::Invoice do
  # ...

  it "sends a draft: numbered, sent now, due after the payment term" do
    sent = draft.send(number(41), NOW).as(Billing::Invoices::Invoice::Sent)
    sent.number.value.should eq("INV-000042")
    sent.sent_at.should eq(NOW)
    sent.due_at.should eq(NOW + 14.days)
    sent.lines.should eq(draft.lines)
    sent.status.should eq(Billing::Invoices::Invoice::Status::Sent)
  end

  # ...
end
```

A store gets a round-trip spec that saves and loads every state, so a field it forgets to map is found ([Domain Models](domain-models.md#testing-models-and-stores)).

## Pages, forms, and agents

Pages are tested as a browser uses them, with CSS selectors on the HTML:

| Helper | Does |
|---|---|
| `client.visit(href)` | Asks as a browser does; HTML is not checked against `openapi.json` |
| `have_element(css, text:, count:)` | Finds elements in a response or an HTML string |
| `client.submit(Command.action(...), form: {...})` | Sends a form body as a browser does |
| `redirect_to(href)`, `have_field_error(name, code)` | A form's 303, and a field marked with an error by its code, in any language |
| `Punch::Spec.render(component)` | Renders one component without a request |
| `client.read(href)`, `client.act(page, title, fields)`, `have_action(title)` | A page and its action blocks, as an agent uses them |
| `client.live(href)`, `have_stream_action(action, target)` | A live page's stream |

See [Pages and Components](pages-and-components.md#testing-pages-and-components), [Forms](forms.md#testing-forms), [Pages for Agents](pages-for-agents.md#testing-pages-as-an-agent), and [Turbo Frames, Streams, and Live Pages](turbo-frames-streams-and-live-pages.md#testing-live-pages).

## Events, jobs, mail, and webhooks

Background work is tested end to end: the Command commits, the outbox hands the job over, the spec runs the due jobs, and checks the mail that went out.

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

| Helper | Checks |
|---|---|
| `->{ … }.should publish(Event, field: value, times: n)` | The block committed such events |
| `->{ … }.should enqueue(Job, field: value, at: time, times: n)` | The block left such jobs waiting, through the outbox or directly |
| `Punch::Spec.perform_jobs` | Runs every due job, and returns them |
| `Punch::Spec.mails` | The mail captured so far, with subject, recipients, and both parts |
| `client.deliver_webhook(path, body, secret:)` | A webhook signed as the provider signs it; a wrong secret or an old time tests the refusals |
| `Punch::Spec.ops` | This example's `ops.db`, for what the ops console would show |

A matcher that names a field the event or job does not have does not compile, so a renamed field breaks the spec instead of letting it pass by accident. See [Domain Events](domain-events.md), [Background Jobs](background-jobs.md), [Mail](mail.md), and [Inbound Webhooks](inbound-webhooks.md).

## Controlling time

An operation that needs the time asks for `clock : Punch::Clock` in `call` and reads `clock.now`, instead of calling `Time.utc`. A spec fixes the clock and moves it:

```crystal source=reference/spec/billing/customers/statement_spec.cr select="customer statement adds up sent invoices per currency, leaving drafts out"
# spec/billing/customers/statement_spec.cr
# ...

describe "customer statement" do
  it "adds up sent invoices per currency, leaving drafts out" do
    client = AccountsSpec.client
    customer = customer_id(client)
    clock = Punch::Spec.clock
    send_invoice(client, customer, 10_000)
    clock.now = Time.utc - 90.days
    send_invoice(client, customer, 5_000)
    clock.now = Time.utc
    send_invoice(client, customer, 700, "CHF")
    client.post(Billing::Invoices::Create.path, json: {customer_id: customer.value, currency: "EUR", lines: [{description: "Draft", quantity: 1, unit_price: 99}]})

    response = client.get(Billing::Customers::Statement.path(customer_id: customer))
    response.status.should eq(HTTP::Status::OK)
    balances(response).should eq([{"CHF", 1, 0, 700}, {"EUR", 2, 1, 15_000}])
    client.get(Billing::Customers::Statement.path(customer_id: InvoicingSpec.parse(Billing::Customers::ID, 999))).status.should eq(HTTP::Status::NOT_FOUND)
  end

  # ...
end
```

`Punch::Spec.clock` returns a clock that moves only when told: `clock.now = …` or `clock.advance(40.days)`. An invoice becomes overdue in a spec because it was sent ninety days ago, not because its row was edited. Sessions and the actor's last-seen time read the same clock.

Punch's own clocks are separate, so a spec moves only what it means to move: `Punch::Spec.job_clock` for when jobs are due, `cache_clock` for cache expiry, and `rate_limit_clock` for rate limits. Never `sleep` in a spec.

`Time.utc` called directly by the app's own code is not controlled. The rule is to ask for the clock instead; the accounts slice that `punch new` writes still calls `Time.utc` in a few places.

## Credentials and settings in specs

`Punch::Spec.credentials(Type, yaml)` in the spec helper sets the app's credentials for every example, as the reference app's helper does above. Use values that are obviously for specs: a key ring that only specs use, `capture` as the mail delivery, a webhook secret defined in the spec support. Real credentials never reach the spec helper.

## What to test at which level

A Punch app gets a lot of checking without a single spec: the compiler checks types, routes, links, forms, translations, policies on every operation, and exhaustive handling of results; `punch check` checks query plans, migrations, the API contract, and payload compatibility. Specs cover what those cannot: behaviour. A regime that works:

| Level | Spec | For |
|---|---|---|
| Model | A plain spec, no database | Every transition and rule of a model, and every boundary of a domain primitive (`schema_agrees`) |
| Store | `round_trip` | Every state saved and loaded |
| Operation | An endpoint spec through the client | Each outcome in `Output`: the success, each failure, and what the policy refuses |
| Page | `client.visit` | Every page rendered at least once; what each viewer sees and does not see |
| Form | `client.submit` | Success, a refusal shown again, and a double submit for idempotent forms |
| Background | `publish`, `enqueue`, `perform_jobs`, `mails` | That the work happens once, after commit, with the right content |
| Journey | Several requests across namespaces | A few paths that matter most: signing up and sending a first invoice, an agent from `/llms.txt` to an action |

Leave out what the framework already guarantees: that a blank `Punch::NonEmptyString` is refused, that an operation without a token answers 401, that a required field is required. Test the app's own decisions: which policy, which result, which rule.

## Common mistakes

**Mocking the database.** There is nothing to mock: each example has its own real database, in memory, and it is fast. A mocked query tests the mock.

**Sleeping in a spec.** `sleep 1.second` to make a job due, a cache expire, or a rate limit reset makes specs slow and flaky. Move the matching clock.

**Calling `Time.utc` in an operation.** The spec cannot control it, and the invoice is overdue only on some days. Ask for `clock : Punch::Clock` in `call`.

**Turning off the contract check because a spec fails.** It cannot be turned off, on purpose. Fix the response, or regenerate `openapi.json` if the change was intended.

**Testing generated code.** A spec that checks a generated query returns its columns tests the generator. Test the behaviour the query serves.

**Hand-written paths.** `client.post("/billing/invoices/1/send")` keeps passing after the route changes, against a 404 the spec may even expect. Use `Send.path(invoice_id: id)`.

**Specs that depend on order.** Examples run in random order, each with an empty database. A spec that relies on data another made fails sometimes. Create what each example needs.

## Limits

- No browser tests. Pages are tested as HTML; JavaScript behaviour needs a browser tool outside Punch.
- No way to control `Time.utc` called directly.
- Specs run against SQLite in memory; behaviour that depends on a second connection uses `Punch::Spec.database(file: true)`.

## Rules

- Give every example what it needs; never rely on another example's data. ([A fresh database per example](#a-fresh-database-per-example))
- Test operations through `Punch::Spec.client`, so the contract check runs. ([Endpoint specs](#endpoint-specs))
- Never turn off or work around the contract check. ([The contract check](#the-contract-check))
- Build paths with `Operation.path` and `Page.href` in specs too. ([Endpoint specs](#endpoint-specs))
- Ask for the clock in `call`, and move clocks in specs; never sleep. ([Controlling time](#controlling-time))
- Test the app's decisions, not the framework's guarantees. ([What to test at which level](#what-to-test-at-which-level))

## Reference

- [Testing](../testing.md): every helper and matcher.
- [Persistence](../persistence.md#testing): databases in specs.
- [JSON endpoints and OpenAPI](../openapi.md): the document the contract check uses.
