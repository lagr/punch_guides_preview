# Domain Models

Applies to Punch 0.1.

In Punch, a model decides and an operation orchestrates. A model holds the rules about one kind of thing: derived values, its own invariants, and which changes of state are allowed. It never receives a database handle. An operation loads the data, asks the model, saves what the model returned, publishes events, and answers. This guide shows the two shapes a model takes, how to choose between them, and how to test them.

After reading this guide, you will know:

- where business rules live, and why models never take a database handle;
- how to add behaviour to a generated record, the default for most entities;
- when an entity needs one type per state, and how to write those types and their store;
- how list rows share a rule with the model without copying it;
- why Punch has no model validations, and where each kind of rule goes instead;
- how to prove that a store saves and loads every state without losing a field.

Before you start: you know how named queries and generated records work (the [queries reference](../queries.md)) and how operations return results ([Results, Errors, and Status Codes](results-errors-and-status-codes.md)). The examples come from the reference app's billing slice: customers are the simple case, invoices the one with a lifecycle.

## A model decides, an operation orchestrates

Here is the reference app's Command that sends an invoice, reduced to what it does:

```crystal source=reference/app/billing/invoices/operations/send.cr select=Billing::Invoices::Send#call
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

`Send` loads the invoice through its store, asks it whether it may be sent (`invoice.send(...)`), saves the result, publishes an event, enqueues a reminder, and answers. It does not know what makes an invoice sendable, how numbers are assigned, or when it falls due. The model knows: `Draft#send` returns either a `Sent` invoice or `NoLines`.

That split has three consequences:

- **Rules are testable without a database.** A model is a plain value: build one in a spec and call its methods.
- **Rules are in one place.** Every operation that sends invoices, and the page that shows the Send button, ask the same method.
- **A model's diff is a rule change.** When `invoice.cr` changes, a business rule changed. Those are the diffs to review most closely.

`Punch/ModelDatabase` keeps the line: a `Punch::Reader`, `Punch::Writer`, or `Punch::Database` anywhere but an operation or a store is reported.

```text
`Punch::Writer` outside an operation or a store: models never take a database handle; load and save in the operation or a Store (lib/punch/docs/models.md)
```

## Two tiers

Punch models come in two tiers:

| | Tier 1: the generated record | Tier 2: state types and a store |
|---|---|---|
| Shape | The row, as `punch db:generate` writes it | One hand-written immutable type per lifecycle state |
| Behaviour | Methods in `record.cr`, which reopens the record | Methods on each state; transitions return the next state |
| Loading and saving | The generated `Table` | A store, with the generated queries |
| Fits | Reference data; things that are created, shown, and edited | Things whose state decides what may happen to them |
| Reference app | Customers | Invoices |

Start with tier 1. Move an entity to tier 2 when one of these holds:

- its state decides which operations are allowed;
- it has rules that span itself and its child rows;
- the same status check has turned up in a second operation.

When unsure, stay with tier 1. Moving up later is mostly contained in one entity's directory.

## Tier 1: behaviour on the generated record

`punch db:generate` writes a `Record` for each mapped table into `records.gen.cr`: one field per column, each already in its domain type (`name : Punch::NonEmptyString`, `email : Punch::Email`), plus a `Table` with `find`, `insert`, `update`, and `delete`. Behaviour goes in a hand-written file beside it that reopens the struct:

```crystal source=reference/app/billing/customers/record.cr
# app/billing/customers/record.cr
# Tier 1 (lib/punch/docs/models.md): a customer is reference data, created, shown, and
# edited. Its behaviour lives on the generated record, reopened here.
struct Billing::Customers::Record
  # The name invoices are addressed to: the company when there is one.
  def display_name : String
    (company_name || name).value
  end
end
```

The file's name is fixed: `Billing::Customers::Record` lives in `app/billing/customers/record.cr`, and the generated file's comment points there. `Punch/RecordFile` reports a reopened record anywhere else, so that pointer is always right.

There is no mapping code at all. The record has the table's shape, and every named query that selects whole rows returns it. Add only methods, and none that take a database handle.

## Tier 2: one type per state

An invoice has a lifecycle. A draft may be revised, deleted, or sent. A sent invoice is final: it has a number and a due date, and it can be paid. A paid one is settled. In tier 1, those rules would be status checks scattered over operations (`if invoice.status == "draft"`). In tier 2, each state is its own type, and each type has only the methods that state allows:

```crystal source=reference/app/billing/invoices/invoice.cr select=Billing::Invoices::Invoice::Draft,Billing::Invoices::Invoice::Sent,Billing::Invoices::Invoice::Any
# app/billing/invoices/invoice.cr
# Tier 2 (lib/punch/docs/models.md): an invoice has a lifecycle, and its state decides what
# may happen to it. Each state is its own immutable type; a transition is a
# method that returns the next state. Invoices::Store turns rows into these
# types and back. Nothing here touches the database.
module Billing::Invoices::Invoice
  # ...

  # Being written: can be revised, sent, or deleted.
  record Draft,
    id : Invoices::ID,
    customer_id : Customers::ID,
    currency : Punch::Currency,
    lines : Array(Line),
    created_at : Time do
    include Billed

    def status : Status
      Status::Draft
    end

    def revise(*, currency : Punch::Currency = @currency, lines : Array(Line) = @lines) : Draft
      copy_with(currency: currency, lines: lines)
    end

    # Issues the invoice under *number*. An invoice without lines cannot
    # be sent.
    def send(number : Invoices::Number, at : Time) : Sent | NoLines
      return NoLines.new if lines.empty?

      Sent.new(id, customer_id, currency, lines, created_at, number: number, sent_at: at, due_at: at + Invoices::PAYMENT_TERM)
    end
  end

  # Issued: numbered, final, and due on a date.
  record Sent,
    id : Invoices::ID,
    customer_id : Customers::ID,
    currency : Punch::Currency,
    lines : Array(Line),
    created_at : Time,
    number : Invoices::Number,
    sent_at : Time,
    due_at : Time do
    include Billed
    include Due

    def status : Status
      Status::Sent
    end

    def paid_at : Time?
      nil
    end

    # Settles the invoice with a payment of *amount* in *currency*, which
    # must be its total in its currency.
    def pay(amount : Punch::Money, currency : Punch::Currency, at : Time, reference : Punch::NonEmptyString) : Paid | WrongAmount
      return WrongAmount.new(total, @currency) unless amount == total && currency == @currency

      Paid.new(id, customer_id, @currency, lines, created_at, number, sent_at, due_at, paid_at: at, payment_reference: reference)
    end
  end

  # ...

  # Every state; `Issued` the ones that are final.
  alias Any = Draft | Sent | Paid
  # ...
end
```

- **Only `Draft` has `send` and `revise`.** Calling `send` on a sent invoice does not compile. The rule "only a draft can be sent" is not checked anywhere; it is the shape of the types.
- **A transition returns the next state** or a reason it cannot happen: `Draft#send` returns `Sent | NoLines`, `Sent#pay` returns `Paid | WrongAmount`. The caller must handle both, because a `case … in` over a union must be exhaustive.
- **States are immutable.** A transition makes a new value; nothing changes in place. `revise` returns a new `Draft`.
- **`Any` is the union of every state.** Loading an invoice gives an `Invoice::Any`, and the operation branches on the type.

An operation then reads like the lifecycle:

```crystal illustrative="the shape of Send#call, quoted in full above"
case invoice = Store.load(db, input.invoice_id)
in Nil                          then NotFound.new
in Invoice::Sent, Invoice::Paid then AlreadySent.new(invoice)
in Invoice::Draft
  # only a draft has `send`
end
```

Adding a state, say `Cancelled`, adds it to `Any`, and every `case … in` that has to handle it stops compiling until it does. That is the point of the tier: the compiler finds every place a new state matters.

> [!NOTE]
> **Coming from Rails:** a state machine gem checks transitions at run time and keeps one class with a `status` column. Here each state is a type, so a transition that is not allowed is a compile error rather than an exception. See [Coming from Rails](coming-from-rails.md).

## Branch on types, not on statuses

Once an entity has state types, operations branch on them and never on the stored status. `Punch/StatusComparison` reports a status comparison inside a Query or Command:

```text
Compares `invoice.status` in an operation: branch on the state types instead (`case invoice in Invoice::Sent`) (lib/punch/docs/models.md)
```

It reports `invoice.status == …`, `row.status.sent?`, `case row.status`, and the like. Turning a stored status into a state belongs in the store; showing a status in a result is fine.

The rule cannot see types, so it also reports a status check on a tier 1 entity. Treat that as the signal: a status check in an operation is the moment to consider tier 2. When it is not worth it yet, disable the rule on that one line with `# ameba:disable Punch/StatusComparison` and a reason, which leaves a mark to come back to.

## The store

A store turns rows into the right state and states back into rows. It is the one place besides operations that takes a database handle, and its name ends in `Store`:

```crystal source=reference/app/billing/invoices/store.cr select=Billing::Invoices::Store#load,Billing::Invoices::Store#save
# app/billing/invoices/store.cr
# Loads and saves Billing::Invoices::Invoice states with the generated queries. Call
# it inside one `read` or `write`, so a load and the save after it see the
# same data (lib/punch/docs/models.md).
module Billing::Invoices::Store
  # ...

  def load(db : Punch::Reader | Punch::Writer, id : ID) : Invoice::Any?
    row = Table.find(db, id) || return
    lines = Queries.lines(db, invoice_id: id).map do |line|
      Invoice::Line.new(line.description, line.quantity, line.unit_price)
    end
    payment = row.status == "sent" ? Queries.payment(db, invoice_id: id) : nil

    case status(row.status, id, payment.try(&.paid_at))
    in .draft?
      Invoice::Draft.new(row.id, row.customer_id, row.currency, lines, row.created_at)
    in .sent?
      Invoice::Sent.new(row.id, row.customer_id, row.currency, lines, row.created_at,
        number: row.number || missing(id, "number"),
        sent_at: row.sent_at || missing(id, "sent_at"),
        due_at: row.due_at || missing(id, "due_at"))
    in .paid?
      payment ||= missing(id, "invoice_payments")
      Invoice::Paid.new(row.id, row.customer_id, row.currency, lines, row.created_at,
        number: row.number || missing(id, "number"),
        sent_at: row.sent_at || missing(id, "sent_at"),
        due_at: row.due_at || missing(id, "due_at"),
        paid_at: payment.paid_at, payment_reference: payment.reference)
    end
  end

  # ...

  # Writes the state and replaces the lines. Organization and customer stay. A
  # payment is written once, when a `Sent` invoice loaded in the same
  # transaction becomes `Paid`; it never changes after.
  def save(db : Punch::Writer, invoice : Invoice::Any) : Nil
    # Numbers come from `next_number` in the same transaction.
    saved = case invoice
            in Invoice::Draft
              Queries.save(db, id: invoice.id, currency: invoice.currency, status: "draft", number: nil, sent_at: nil, due_at: nil)
            in Invoice::Sent
              Queries.save(db, id: invoice.id, currency: invoice.currency, status: "sent", number: invoice.number, sent_at: invoice.sent_at, due_at: invoice.due_at)
            in Invoice::Paid
              # Nothing of the sent invoice changes; only the payment is new.
              Queries.record_payment(db, invoice_id: invoice.id, reference: invoice.payment_reference, amount: invoice.total, paid_at: invoice.paid_at)
              return
            end

    raise Punch::Database::Error.new("invoice #{invoice.id} is gone") if Punch.cannot_violate(saved).zero?
    Queries.delete_lines(db, invoice_id: invoice.id)
    insert_lines(db, invoice.id, invoice.lines)
  end

  # ...
end
```

- **`load`** reads the row, its lines, and its payment, and builds a `Draft`, `Sent`, or `Paid`. A sent invoice without a number is not a state the model allows, so a row like that raises a `Punch::Database::RowError` naming the column: it is a bug in the data, not an outcome.
- **`save`** writes the state back with the generated queries. A paid invoice adds its payment row and changes nothing else.
- **Load, change, and save happen in one write transaction**, the Command's. SQLite runs write transactions one after another, so nothing can change the invoice between the load and the save. No version column and no row lock are needed.

A state does not need a status of its own in the table. The reference app's `Paid` is a sent invoice with a row in `invoice_payments`; the store decides which state a row is.

## Lists share rules through behaviour modules

A list does not load full models. The invoice list reads one narrow row per invoice, with the customer's name and the total computed in SQL. But the list needs a rule the model has: whether an invoice is overdue. Copying the rule would let the list and the detail page disagree.

So the rule lives in a module that names the fields it needs as abstract methods:

```crystal source=reference/app/billing/invoices/invoice.cr select=Billing::Invoices::Invoice::Due
# app/billing/invoices/invoice.cr
# Tier 2 (lib/punch/docs/models.md): an invoice has a lifecycle, and its state decides what
# may happen to it. Each state is its own immutable type; a transition is a
# method that returns the next state. Invoices::Store turns rows into these
# types and back. Nothing here touches the database.
module Billing::Invoices::Invoice
  # ...

  # Whether an invoice is unpaid past its due date. Included by `Sent`,
  # `Paid`, and the rows of the list queries (`-- include
  # Billing::Invoices::Invoice::Due`), so lists and details share one rule; a draft
  # row has no due date and is never overdue.
  module Due
    abstract def due_at : Time?
    abstract def paid_at : Time?

    def overdue?(now : Time) : Bool
      due = due_at
      due && paid_at.nil? ? now > due : false
    end
  end

  # ...
end
```

`Sent` and `Paid` include it, and so does the list's row: the named query says so with an `-- include` line in its header:

```sql illustrative="the header of Billing::Invoices::Queries.page, shortened"
-- name: Page :page
-- One organization's invoices with their customer's name and total, a
-- page at a time in the order they were created; …
-- include Billing::Invoices::Invoice::Due
SELECT invoices.id, invoices.status, …, invoices.due_at, invoice_payments.paid_at
```

The generated row then includes `Due`, and `row.overdue?(now)` is the same rule as `sent.overdue?(now)`. A row that lacks one of the module's fields, or has one of another type, does not compile. Give such a module only methods computed from its fields; like a model, it never touches the database.

## Where each kind of rule goes

Punch has no model validations. A Rails model collects every kind of check in `validates` lines and runs them when it saves. Punch puts each kind of rule where it can be enforced best:

| Rule | Home | Example |
|---|---|---|
| What a single value may be | Its type, checked when the request is parsed | `Punch::Email`, `Quantity` (1 to 10,000) |
| How fields of one request relate | The input's `check` | At most 100 lines per invoice |
| What may happen to a thing in its state | The model's types and transitions | Only a draft can be sent |
| What follows from the thing's own data | A model method | A sent invoice is overdue after its due date |
| Uniqueness | A unique index; the violation comes back as a typed result | One customer per email address in an organization |
| Rules across rows or entities | The Command, inside its transaction | The customer must belong to the actor's organization |
| Derived values needed in lists | SQL in a named query | The invoice total in the list |

Uniqueness is the clearest case. Checking "is this email taken?" before inserting is a race: two requests can both pass the check. The database's unique index decides instead, and the generated insert returns `EmailTaken` as a value the Command passes on, a 422 at `email` ([Results, Errors, and Status Codes](results-errors-and-status-codes.md#field-errors-rejections-and-violations)).

By the time a model gets a value, the value is valid. A model checks only what depends on its own state.

## Testing models and stores

Models and rows are plain structs, so their specs need no database. The reference app checks that the list row and the model agree on "overdue":

```crystal source=reference/spec/billing/invoices/invoice_spec.cr select="Billing::Invoices::Invoice shares the overdue rule with list rows, where a draft has no due date"
# spec/billing/invoices/invoice_spec.cr
# ...

describe Billing::Invoices::Invoice do
  # ...

  it "shares the overdue rule with list rows, where a draft has no due date" do
    sent = draft.send(number, NOW).as(Billing::Invoices::Invoice::Sent)
    paid = sent.pay(sent.total, EUR, NOW + 20.days, InvoicingSpec.parse(Punch::NonEmptyString, "pay_1")).as(Billing::Invoices::Invoice::Paid)
    row = ->(due_at : Time?, paid_at : Time?) do
      Billing::Invoices::Queries::Page::Row.new(sent.id, "sent", nil, EUR, sent.customer_id,
        InvoicingSpec.parse(Punch::NonEmptyString, "Ada"), 0_i64, nil, due_at, paid_at)
    end
    [NOW + 14.days, NOW + 14.days + 1.second, NOW + 30.days].each do |now|
      row.call(sent.due_at, nil).overdue?(now).should eq(sent.overdue?(now))
      row.call(paid.due_at, paid.paid_at).overdue?(now).should eq(paid.overdue?(now))
    end
    row.call(nil, nil).overdue?(NOW + 1000.days).should be_false
  end

  # ...
end
```

A store needs one more kind of spec. A field the store forgets to save or load is a quiet bug: everything compiles, and the field comes back empty. `Punch::Spec.round_trip` saves and loads each state and compares, field by field:

```crystal source=reference/spec/billing/invoices/store_spec.cr select="Billing::Invoices::Store loads what it created and saved, in every state"
# spec/billing/invoices/store_spec.cr
# ...

describe Billing::Invoices::Store do
  it "loads what it created and saved, in every state" do
    Punch::Spec.db.write do |db|
      customer = BillingSpec.customer(db)
      lines = [BillingSpec.line("Consulting", 3, 12_500), BillingSpec.line("Travel", 1, 4_990)]
      draft = Billing::Invoices::Store.create(db, organization_id: customer.organization_id, customer_id: customer.id, currency: EUR, lines: lines, at: Time.utc)
      Billing::Invoices::Store.load(db, draft.id).should eq(draft)
      revised = draft.revise(currency: InvoicingSpec.parse(Punch::Currency, "CHF"), lines: lines.reverse)
      sent = revised.send(Billing::Invoices::Store.next_number(db), Time.utc).as(Billing::Invoices::Invoice::Sent)
      paid = sent.pay(sent.total, sent.currency, Time.utc, InvoicingSpec.parse(Punch::NonEmptyString, "pay_1")).as(Billing::Invoices::Invoice::Paid)

      Punch::Spec.round_trip(revised, sent, paid, covering: Billing::Invoices::Invoice::Any) do |invoice|
        Billing::Invoices::Store.save(db, invoice)
        Billing::Invoices::Store.load(db, invoice.id)
      end
      Billing::Invoices::Store.next_number(db).value.should eq("INV-000002")
    end
  end

  # ...
end
```

It fails when a state comes back different, naming the field (`Round trip of Billing::Invoices::Invoice::Sent changed due_at: saved …, loaded …`), when a load returns `nil` or another state, and, with `covering:`, when a state of the union has no example. Give every field a value other than its default, or a forgotten field comes back equal by chance.

## Common mistakes

**Passing the database to a model.** `def send(db : Punch::Writer)` on `Draft` lets the model load and save, and `Punch/ModelDatabase` reports it. Load in the operation or the store, pass the model what it needs as values, and save what it returns.

**Checking a status in an operation.** `return AlreadySent.new if invoice.status.sent?` works until a fourth state appears and the check is wrong in a place nobody remembers. `Punch/StatusComparison` reports it. Move the entity to state types, or disable the rule on that line with a reason while you decide.

**Looking up before inserting.** A Command that runs `Table.find_by_email` and then inserts has a race between the two. Insert, and handle the violation the generated insert returns.

**Mutating a state in place.** Giving `Draft` a `property` and assigning to it hides a transition. Return a new value from a method named for the change (`revise`).

**Rules in `record.cr` that need other rows.** A method on the customer's record that totals the customer's invoices would need a database handle. A value computed across rows is a named query, or a Query operation; a rule across entities belongs in the Command.

**Testing a store only with drafts.** The round trip passes, and the sent state's `due_at` is never saved. Pass every state, and `covering:` the union, so a new state without an example fails the spec.

## Limits

- The `-- include` header is still open to change: it may give way to reopening the generated row in a hand-written file, as records are.
- There is no built-in state machine helper. State types are plain Crystal records, and that is the design.
- Rules across several entities have no home of their own beyond the Command and its transaction.

## Rules

- A model never takes a database handle; operations and stores load and save. ([A model decides, an operation orchestrates](#a-model-decides-an-operation-orchestrates))
- Start with behaviour on the generated record; move to state types when state decides what may happen. ([Two tiers](#two-tiers))
- Put a record's behaviour in its `record.cr`, and nowhere else. ([Tier 1: behaviour on the generated record](#tier-1-behaviour-on-the-generated-record))
- Make each state a type with only the transitions it allows; transitions return the next state or a reason. ([Tier 2: one type per state](#tier-2-one-type-per-state))
- Branch on state types in operations, never on a stored status. ([Branch on types, not on statuses](#branch-on-types-not-on-statuses))
- Share a rule between models and list rows through a behaviour module, never by copying it. ([Lists share rules through behaviour modules](#lists-share-rules-through-behaviour-modules))
- Put each rule where it is enforced best: types, `check`, the model, a unique index, or the Command. ([Where each kind of rule goes](#where-each-kind-of-rule-goes))
- Round-trip every state through its store in a spec. ([Testing models and stores](#testing-models-and-stores))

## Reference

- [Domain models](../models.md): outlines, both tiers, behaviour modules, the lint rules, and `round_trip`.
- [Queries](../queries.md): generated records, named queries, and violations.
- [Testing](../testing.md): the spec helpers.
