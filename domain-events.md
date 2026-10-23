# Domain Events

Applies to Punch 0.1.

When an invoice is sent, more than the invoice changes: the customer's balance is out of date, the customer should get a mail, a page someone has open should show the new state. If `Send` did all of that itself, it would grow with every new reaction, and nobody reading it could tell its job from its side effects. In Punch, a Command announces what happened as a typed event, and code elsewhere reacts to it. The compiler knows every publisher and every subscriber, so the side effects stay visible: `punch events` prints them all.

After reading this guide, you will know:

- how to announce what a Command did, and react to it elsewhere;
- which of the three deliveries to choose for a reaction;
- why nothing is delivered for a Command that failed;
- why the core of a use case must not hide in a subscriber;
- how to see the whole graph, and how to test it.

Before you start: you know Commands and their results ([Results, Errors, and Status Codes](results-errors-and-status-codes.md)), and what a job is ([Background Jobs](background-jobs.md)).

## An event is a fact

An event is a struct that includes `Punch::Event`, named in the past tense for what happened. It holds the IDs and small values a subscriber needs to look up the rest. The reference app's events are in its billing slice's `events.cr`:

```crystal source=reference/app/billing/events.cr select=Billing::BalanceChanged,Billing::InvoiceSent
# app/billing/events.cr
module Billing
  # Included by every event that changes what a customer owes, so one
  # subscriber hears them all (`Customers::ForgetStatement`).
  module BalanceChanged
    abstract def customer_id : Customers::ID
  end

  # ...

  # An invoice was issued to its customer: numbered, final, and due.
  # Published by `Invoices::Send` once the invoice is saved.
  struct InvoiceSent
    include Punch::Event
    include BalanceChanged
    include InvoiceChanged

    getter invoice_id : Invoices::ID
    getter customer_id : Customers::ID

    def initialize(@invoice_id, @customer_id)
    end
  end

  # ...
end
```

An event is immutable: a class, or a setter, does not compile. Several events can share a module, such as `BalanceChanged` here, so that one subscriber hears every event that changes what a customer owes.

## Publishing

Only a Command publishes, and only the events its `call` declares. It asks for `events : Punch::Events(…)`, listing every event it may publish, and calls `publish`:

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

That type is the declaration. Publishing an event it does not list does not compile, and neither does a Query or Page that asks for `Punch::Events`. Never make a handle any other way: `punch events` could not see the event, and `punch check` reports it.

An event is part of the Command's transaction. If the Command returns a failure or raises, its writes roll back and its events are never delivered. A subscriber never reacts to something that did not happen.

## Three deliveries

A subscriber includes `Punch::Subscriber(Event)` and defines `call`. How and when it runs is its delivery:

| Delivery | Runs | For | `call` |
|---|---|---|---|
| after commit (the default) | once the Command has committed, before the response | fast reactions to committed data: forgetting a cached value, updating open pages | `call(event) : Nil` |
| deferred | as a job, through the outbox, after the commit | anything slow, retryable, or outside the app: mail, calls to other systems | `call(event, job : Punch::Job::Context) : Punch::Job::Result` |
| inline | inside `publish`, in the Command's transaction | the rare projection that must stay consistent with the data | `call(event, db : Punch::Writer) : Nil` |

### After commit

The default. The reference app keeps each customer's statement in a cache, and a sent or paid invoice makes it stale:

```crystal source=reference/app/billing/customers/subscribers/forget_statement.cr
# app/billing/customers/subscribers/forget_statement.cr
module Billing::Customers
  # A sent or paid invoice changes what its customer owes, so the cached
  # statement is forgotten once the change is committed; the next request
  # computes it again. Without this, the statement would be up to a minute
  # old. Punch writes the `call` (`punch events` shows the store).
  @[Punch::Subscriber::Forget(STATEMENTS, key: customer_id)]
  struct ForgetStatement
    include Punch::Subscriber(BalanceChanged)
  end
end
```

For forgetting cache entries, `@[Punch::Subscriber::Forget]` writes the subscriber's `call` itself ([Caching](caching.md)). An after-commit subscriber that raises does not fail the Command, whose data is committed already: the exception is logged and counted, and the other subscribers and the response go on. Keep these fast, since the client waits for them.

### Deferred

A deferred subscriber is a job. Its entry is written to the outbox in the Command's transaction, one per subscriber and event, so the job exists exactly when the data does. A worker runs it after the commit, with retries. The reference app mails a sent invoice this way:

```crystal source=reference/app/billing/invoices/subscribers/mail_invoice.cr
# app/billing/invoices/subscribers/mail_invoice.cr
module Billing::Invoices
  # Mails a sent invoice to its customer. Deferred (lib/punch/docs/events.md): a job
  # through the outbox, so the mail goes out exactly when the invoice was
  # sent, after the response, and is retried if delivery fails.
  #
  # The mail is in the customer's language, else in the organization's,
  # else in the app's default.
  @[Punch::Subscriber::Delivery(:deferred)]
  @[Punch::Job::Options(queue: "mail")]
  struct MailInvoice
    include Punch::Subscriber(InvoiceSent)

    def call(event : InvoiceSent, job : Punch::Job::Context) : Punch::Job::Result
      invoice, customer, organization = job.read do |db|
        customer = Customers::Table.find(db, event.customer_id)
        {Store.load(db, event.invoice_id), customer, customer.try { |found| Accounts::Organizations::Table.find(db, found.organization_id) }}
      end
      return discard("invoice #{event.invoice_id} is gone") unless invoice.is_a?(Invoice::Issued) && customer

      expires_at = job.now.at_beginning_of_second + Sharing::VALID
      link = SharedPage.href(token: Sharing.token(invoice, expires_at))
      locale = Punch::I18n.locale((customer.locale || organization.try(&.locale)).try(&.to_s))
      mail = InvoiceMail.new(invoice, customer, link, Invoicing::Credentials.current.issuer.name.value)
      Punch::Mailer.deliver(mail, locale: locale, job: job)
    end
  end
end
```

Everything about jobs applies: it runs at least once, so make it safe to repeat; it takes `@[Punch::Job::Options]` and `@[Punch::Job::Concurrency]`; it returns `retry` or `discard` for expected outcomes ([Background Jobs](background-jobs.md)). The event is the job's payload, so its shape is recorded in `payloads.lock.json`, and a breaking change to it raises the event's version like a job's ([Background Jobs: changing a payload](background-jobs.md#changing-a-payload)).

### Inline

An inline subscriber writes in the Command's own transaction. If it raises, the Command fails and everything rolls back. It is for a projection that must never disagree with the data, such as a per-customer total kept in a table:

```crystal illustrative="the shape of an inline subscriber; the reference app has none"
@[Punch::Subscriber::Delivery(:inline)]
struct CountSent
  include Punch::Subscriber(Billing::InvoiceSent)

  def call(event : Billing::InvoiceSent, db : Punch::Writer) : Nil
    Queries.count_sent(db, customer_id: event.customer_id)
  end
end
```

Prefer the other two. An inline subscriber adds work to every Command that publishes the event, and a failure in it fails them all.

## Core flow stays explicit

A subscriber is for a reaction the Command does not need to know about. A rule the Command must act on is not a subscriber. If sending must check the customer's credit, `Send` calls that check and returns its failure; it does not publish an event and hope a subscriber refuses.

Two limits keep the flow readable: subscribers do not publish events themselves, and a deferred subscriber's writes publish nothing. So a reader can always follow a use case from its Command, one step out to its subscribers, and no further.

## The graph

`punch events` builds the app and prints every event, who publishes it, and who listens:

```console
$ punch events
...
Billing::InvoicePaid
  published by  Billing::Payments::Provider#process
  after_commit  Billing::Customers::ForgetStatement

Billing::InvoiceSent
  published by  Billing::Invoices::Send
  after_commit  Billing::Customers::ForgetStatement
  deferred      Billing::Invoices::MailInvoice
...
```

`--json` gives the same as data. An event that nothing publishes is dead code, and so are its subscribers: `punch check` fails on it (its `events` step).

## Testing events

The `publish` matcher checks what a block published, and its failure lists what was published instead. The reference app's spec of the forgotten statement:

```crystal source=reference/spec/billing/customers/forget_statement_spec.cr select="customer statement forgets the statement when an invoice to the customer is sent"
# spec/billing/customers/forget_statement_spec.cr
# ...

describe "customer statement" do
  it "forgets the statement when an invoice to the customer is sent" do
    client = AccountsSpec.client
    customer = customer_id(client)
    send_invoice(client, customer, 1_000)
    path = Billing::Customers::Statement.path(customer_id: customer)
    first = client.get(path)
    etag = first.headers["ETag"]

    -> { send_invoice(client, customer, 2_000) }.should publish(Billing::InvoiceSent, customer_id: customer, times: 1)
    Billing::Customers::STATEMENTS.cached?(customer).should be_false
    now = client.get(path, headers: HTTP::Headers{"If-None-Match" => etag})
    now.status.should eq(HTTP::Status::OK)
    balances(now).should eq([{"EUR", 2, 0, 3_000}])
  end
end
```

`Punch::Spec.published(Event) { … }` returns the events themselves, for a closer look. A failed Command delivers nothing, so its events are not there. Test a subscriber's logic by calling it, `ForgetStatement.new.call(event)`, and run deferred subscribers with `Punch::Spec.perform_jobs` like any job. To call a Command directly, get its handle with `Punch::Spec.events(Command, writer)` ([Domain events: specs](../events.md#specs)).

## Common mistakes

**Doing the side effect in the Command.** A Command that mails, clears caches, and updates a counter in `call` hides each of them in one long method. Publish an event and give each reaction its subscriber.

**Putting core flow in a subscriber.** A subscriber that refuses, or that the use case depends on, hides the use case. Call the rule from the Command.

**Mailing from an after-commit subscriber.** It runs before the response and is not retried. Make it deferred.

**A whole record in an event.** A deferred subscriber would then see the row as it was. Put IDs in the event and read the rest when the subscriber runs.

**An event nobody publishes.** `punch check` fails on its `events` step. Publish it from a Command, or delete it and its subscribers.

## Limits

- Subscribers do not publish events, and a deferred subscriber's writes publish nothing.
- After-commit subscribers get no database handle.
- Events are delivered within the app; publishing to other systems is a deferred subscriber's job.

## Rules

- Name events in the past tense, as immutable structs with IDs and small values. ([An event is a fact](#an-event-is-a-fact))
- Publish only from a Command, through `events : Punch::Events(…)` in `call`. ([Publishing](#publishing))
- After commit for fast reactions, deferred for anything slow, retryable, or outside, inline only for a projection that must stay consistent. ([Three deliveries](#three-deliveries))
- Make deferred subscribers safe to repeat. ([Deferred](#deferred))
- Keep rules the Command must act on in the Command. ([Core flow stays explicit](#core-flow-stays-explicit))

## Reference

- [Domain events](../events.md): every delivery's details, ordering, tracing, and spec helpers.
- [Background jobs](../jobs.md): the outbox and payloads.
- [Cache](../cache.md): `@[Punch::Subscriber::Forget]`.
