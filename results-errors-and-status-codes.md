# Results, Errors, and Status Codes

Applies to Punch 0.1.

An operation can end in more than one way. Sending an invoice can work, find no such invoice, find it already sent, or find it has no lines. In Punch, each of those outcomes is a type, and `call` returns a union of them. The compiler makes every caller handle every outcome, the HTTP status comes from the type, and a failed Command leaves nothing written. Exceptions are kept for what nobody expected: bugs.

After reading this guide, you will know:

- how to model every outcome of an operation as a type, and what the client receives for each;
- why a Command that fails leaves nothing written;
- how a unique index becomes a typed result and a field error, and why rescuing the database's exception is refused;
- where the messages in error responses come from;
- what happens to an exception.

Before you start: you know how operations and their inputs work ([Operations and Routing](operations-and-routing.md), [Input and Domain Types](input-and-domain-types.md)).

## Every outcome is a type

The reference app's `Send` returns one of four types:

```crystal source=reference/app/billing/invoices/operations/send.cr select=Billing::Invoices::NoLines,Billing::Invoices::Send::Output,Billing::Invoices::Send#call
# app/billing/invoices/operations/send.cr
module Billing::Invoices
  # An invoice without lines cannot be sent.
  @[Punch::Status(422)]
  struct NoLines
    include Punch::Problem
  end

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

`Detail` is the sent invoice; `NotFound`, `AlreadySent`, and `NoLines` are the ways sending can fail. The two that other operations return too live in the namespace's `results.cr`:

```crystal source=reference/app/billing/invoices/results.cr select=Billing::Invoices::NotFound,Billing::Invoices::AlreadySent
# app/billing/invoices/results.cr
module Billing::Invoices
  # ...

  @[Punch::Status(404)]
  struct NotFound
    include Punch::Problem
  end

  # A sent invoice is final, also once it is paid.
  @[Punch::Status(409)]
  struct AlreadySent
    include Punch::Problem

    getter number : Number
    getter sent_at : Time

    def initialize(invoice : Invoice::Issued)
      @number = invoice.number
      @sent_at = invoice.sent_at
    end
  end
end
```

Because the result is a union, code that calls `Send` handles it with `case … in`, and the compiler checks that every member has a branch. Add a fifth outcome later, and every caller that does not handle it stops compiling, with the type it misses. A page that shows the result, a job, or a spec cannot forget a case.

> [!NOTE]
> **Coming from Rails:** there is no `raise ActiveRecord::RecordNotFound` caught by a `rescue_from`. "Not found" is a value the operation returns, and its status is declared on its type.

## Statuses come from the types

Each result type's status comes from its `@[Punch::Status(code)]` annotation:

| `call` returns | The client gets |
|---|---|
| `nil` | 204, no body |
| a type with a status below 400 | the type's JSON, `application/json` |
| a type with a status from 400 on | Problem Details, `application/problem+json` |

A failure type includes `Punch::Problem`. That makes it serializable, gives a failure without fields an empty constructor (`NotFound.new`), and requires a status from 400 to 599.

When a union holds several types of the app's, every member must state its status, the success too: `Detail` carries `@[Punch::Status(200)]`. A failure that forgot its annotation would otherwise answer 200, commit the Command's writes, and appear as a success in `openapi.json`. The build refuses it, at the operation's `call`:

```crystal fails=spec/fail/status_missing_in_union.cr
module Notes
  @[Punch::Status(200)]
  struct Added
    include JSON::Serializable

    def initialize
    end
  end

  # The failure forgot its status: it would answer 200 and commit.
  struct Missing
    include JSON::Serializable

    def initialize
    end
  end

  @[Punch::Public]
  @[Punch::RateLimitExempt("a fixture")]
  struct Create < Punch::Command
    struct Input
      include Punch::Input
    end

    def call(input : Input, db : Punch::Writer) : Added | Missing
      rand < 0.5 ? Added.new : Missing.new
    end
  end
end

Punch::Routing::Router.new(Punch::Database.memory)
```

```text
Error: Notes::Create#call can return Notes::Missing, which has no @[Punch::Status], in a result of 2 types of the app's: a type without one answers 200, so a failure would commit its writes and show as a success. Give every member a status: @[Punch::Status(200)] on the success, a status from 400 on for a failure, or `include Punch::Problem` in the failure (lib/punch/docs/operations.md, lib/punch/docs/openapi.md).
```

Declare what `call` returns as the operation's `Output`, `alias Output = Detail | NotFound | AlreadySent | NoLines` right above `call`, which returns `: Output`. Every operation names its answers the same way, in the same place, and `Punch/OperationOutput` reports one that does not. The OpenAPI document lists one response per member of `Output`, so the declaration is the contract, even for an outcome the body does not produce yet ([JSON endpoints and OpenAPI](../openapi.md#the-openapi-document)).

## Problem Details

Every error response has one shape, RFC 9457's Problem Details, whether it comes from a result type, a rejected input, or Punch itself. `AlreadySent` answers:

```json illustrative="the shape of the answer; request_id and the times vary"
{"type": "about:blank", "title": "Conflict", "status": 409, "code": "already_sent",
 "request_id": "01J…", "detail": "This invoice was already sent.",
 "number": "INV-000001", "sent_at": "2026-10-23T09:30:00Z"}
```

- `code` is the type's name in snake_case: `AlreadySent` is `already_sent`. Punch's own problems use the status: `not_found`, `method_not_allowed`, `payload_too_large`, `internal_server_error`.
- The type's fields are added beside the standard members, so a client sees which invoice number was already issued.
- `detail` is the problem's message in the request's language, when the app has one for the code.
- A field named like a standard member (`type`, `status`, `code`, `detail`, `errors`, …) does not compile.

Clients branch on `code`, never on `detail` or `title`. Renaming a problem type's last segment renames its code, which is a breaking change for clients. Moving the type to another file or namespace is not.

## Field errors: rejections and violations

Some failures belong to a field of the input. A rejected input is one ([What a refused input looks like](input-and-domain-types.md#what-a-refused-input-looks-like)): 422, `code` `invalid_input`, and an `errors` list with each error's path and code.

A Command can return the same shape for rules it checks itself, as a `Punch::Rejection`. The reference app's `Customers::Create` does this for a language the app does not offer ([Input and Domain Types](input-and-domain-types.md#parse-at-the-boundary-trust-inside)).

### Violations: what only the database can decide

"This email address is already taken" is a rule no type can check: it depends on every other row. In Punch, the unique index decides it. For each unique index of a mapped table, `punch db:generate` writes a violation type into the entity's `records.gen.cr`, such as `Billing::Customers::EmailTaken`, and the generated writes that can break the index return it instead of raising. `Customers::Create` hands it straight back:

```crystal source=reference/app/billing/customers/operations/create.cr select=Billing::Customers::Create::Output,Billing::Customers::Create#call
# app/billing/customers/operations/create.cr
module Billing::Customers
  # ...

  # Adds a customer to the organization the actor works in. Each
  # organization's customers have unique email addresses.
  @[Punch::Authorize(Accounts::Organizations::Current)]
  struct Create < Punch::Command
    # ...

    alias Output = Created | EmailTaken | Punch::Rejection

    def call(input : Input, db : Punch::Writer, actor : Invoicing::Actor, _grant : Punch::Grant(Create), clock : Punch::Clock) : Output
      if rejection = Invoicing::Languages.check(input.locale)
        return rejection
      end

      case created = Table.insert(db, organization_id: actor.organization_id, name: input.name, company_name: input.company_name, email: input.email,
        created_at: clock.now, locale: input.locale)
      in Record     then Created.new(Customer.new(created))
      in EmailTaken then created # the unique index decides; 422 at `email`
      end
    end
  end
end
```

A violation answers 422 with the code `taken` at its field, like a rejected input:

```crystal source=reference/spec/billing/customers/create_spec.cr select="customers API refuses a second customer with the same address, in any case"
# spec/billing/customers/create_spec.cr
# ...

describe "customers API" do
  # ...

  it "refuses a second customer with the same address, in any case" do
    client = AccountsSpec.client
    create(client)
    response = create(client, "ADA@example.org")
    response.status.should eq(HTTP::Status::UNPROCESSABLE_ENTITY)
    response.json["errors"].should eq(JSON.parse(%([{"path": ["email"], "code": "taken", "params": {}, "message": "Is already taken."}])))
  end

  # ...
end

# ...
```

There is no lookup before the insert, and none is needed. A Command holds SQLite's single write lock, and the index decides: a check first would be slower and still not more correct. On a form, the violation shows as an error at the email field, without any code in the page ([Views and pages](../views.md#forms)).

When the input names the field differently from the column, return `violation.rejection("contact", "email")` with the input's path. When a write cannot violate an index by construction, such as positions the code numbers itself, `Punch.cannot_violate(write)` returns the write's result without the violation, and raises if one happens anyway, because then the construction was wrong ([Queries: violations](../queries.md#violations)).

## A failed Command writes nothing

When a Command returns a failure, meaning a type with a status from 400 on, a rejection, or a violation, Punch rolls its transaction back instead of committing it. Every row it wrote, every event it published, and every job it enqueued is gone.

The reference app checks this for a draft without lines. `Send` takes the next invoice number before it finds there are no lines, and the next invoice still gets the first number:

```crystal source=reference/spec/billing/invoices/send_spec.cr select="invoices API does not send a draft without lines, and numbers nothing"
# spec/billing/invoices/send_spec.cr
# ...

describe "invoices API" do
  # ...

  it "does not send a draft without lines, and numbers nothing" do
    client = AccountsSpec.client
    customer = customer_id(client)
    empty = draft(client, customer, [] of NamedTuple(description: String, quantity: Int32, unit_price: Int32))
    response = client.post(Billing::Invoices::Send.path(invoice_id: empty), headers: Punch::Spec.idempotency_key)
    {response.status, response.json["code"]}.should eq({HTTP::Status::UNPROCESSABLE_ENTITY, "no_lines"})
    client.post(Billing::Invoices::Send.path(invoice_id: draft(client, customer)), headers: Punch::Spec.idempotency_key).json["number"].should eq("INV-000001")
  end
end

# ...
```

So write a Command in the order that reads best, and return the failure where it is found. You never undo half of a Command by hand.

## Where the messages come from

Codes are for programs; messages are for people, and they are rendered from codes at the edge, in the reader's language. The app's locale files hold them:

```yaml source=reference/config/locales/en.yml select=en.problems
# config/locales/en.yml
en:
  # ...
  problems:
    already_sent: "This invoice was already sent."
    invalid_signature: "The webhook's signature does not match."
    no_lines: "A draft needs at least one line before it is sent."
    not_sent: "This invoice has not been sent yet."
    unknown_customer: "There is no such customer."
    unknown_invoice: "There is no such invoice."
    wrong_amount: "The amount does not match the invoice's total."
```

- `problems.<code>` is the message of a whole problem, the `detail` of its response. A failure type's scalar fields can fill it in: `%{number}`.
- `errors.<code>` is the message of a field error, filled in from its parameters: `too_many: "Holds more than %{max}."`.
- Punch's own codes have English and German messages in Punch, and an app can override any of them.

`punch check` (its `i18n` step) fails when a problem type has no message in the app's default language, so a new failure type cannot ship without one ([Translations](../translations.md)).

## Exceptions are bugs

An exception that escapes `call` is not a result. Punch rolls the transaction back, answers 500 with a Problem Details body that carries only the status, the code `internal_server_error`, and the request ID, and never shows the exception to the client. The exception goes to the log with its backtrace, marks the request's trace as failed, and is grouped with others like it in the ops console at `/_punch`, where an operator can find it by the request ID the client saw ([Ops data](../ops.md)).

So raise only where something is wrong with the code or its environment: a broken invariant, a missing table, a disk that is full. Never raise for an outcome a user can cause.

## Common mistakes

**Raising for an expected failure.** `raise NotFoundError` gives the client a 500 and an operator an alert. Return a `NotFound` result with `@[Punch::Status(404)]`.

**Checking for a duplicate before inserting.** A `SELECT` for the email before the insert is extra work, and the index has to decide anyway. Insert, and return the violation.

**Rescuing the database's constraint error.** `rescue Punch::Database::ConstraintError` hides foreign key and `CHECK` failures that are bugs, and `punch check` reports it (`Punch/ConstraintRescue`). Unique indexes come back as violations; check foreign keys in the Command before writing.

**A failure without a status.** In a union, the build refuses it. Include `Punch::Problem` in every failure type.

**Branching on the message.** A client that compares `detail` to a sentence breaks when the wording or the language changes. Compare `code`.

**Returning a database row.** A generated `Record` or `Row` has no JSON form, and returning one does not compile. Return a result struct with the fields to show, built from the row, so the wire shape is declared ([JSON endpoints and OpenAPI](../openapi.md#a-database-row-is-not-a-response)).

## Limits

- A violation names one field, or the input as a whole for an index over several columns.
- Foreign key, `CHECK`, and `NOT NULL` failures still raise: SQLite does not say which foreign key failed. Check what they guard in the Command before writing.
- Problem types are matched to messages by their code alone; two types with the same last name share a code.

## Rules

- Return every expected outcome as a type; raise only for bugs. ([Every outcome is a type](#every-outcome-is-a-type))
- Give every member of a result union a status; failures include `Punch::Problem`. ([Statuses come from the types](#statuses-come-from-the-types))
- Declare `alias Output = …` above `call`, and have `call` return `: Output`. ([Statuses come from the types](#statuses-come-from-the-types))
- Let the unique index decide uniqueness, and return its violation; never look up first, and never rescue `ConstraintError`. ([Violations: what only the database can decide](#violations-what-only-the-database-can-decide))
- Return a failure where it is found; the transaction rolls back by itself. ([A failed Command writes nothing](#a-failed-command-writes-nothing))
- Give every problem a message in `problems.<code>`; clients branch on `code`. ([Where the messages come from](#where-the-messages-come-from))

## Reference

- [JSON endpoints and OpenAPI](../openapi.md): statuses, Problem Details, the OpenAPI document, and breaking changes.
- [Queries: violations](../queries.md#violations): which writes return which violation.
- [Core types](../core-types.md#results-are-unions): results as unions.
- [Translations](../translations.md): codes, messages, and locale files.
