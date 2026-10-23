# Inbound Webhooks

Applies to Punch 0.1.

Other systems tell an app what happened by posting to it: a payment succeeded, a mail bounced, a subscription ended. Such a delivery must be accepted quickly, checked for a forged signature, processed exactly once even when the sender retries, and never lost when processing fails. Punch does this for every provider an app declares: its endpoint stores the request and answers at once, and a job processes it later.

After reading this guide, you will know:

- how to accept a provider's deliveries safely;
- how a delivery's body becomes a typed event your code handles;
- what happens on a bad signature, a duplicate, a failure, and an event you do not handle;
- how to rotate a provider's secret;
- how to inspect, retry, and test deliveries.

Before you start: you know Commands and their transactions ([Operations and Routing](operations-and-routing.md)), and how jobs retry ([Background Jobs](background-jobs.md)). The examples come from the reference app's payment provider, a stand-in for a real one.

## Receive first, process later

Every delivery goes through two steps:

1. **Receive.** `POST /webhooks/<name>` checks the signature, stores the request as it arrived, and answers 204. Storing the request and enqueuing its processing happen in one transaction, through the outbox, so a delivery that was answered is never lost.
2. **Process.** A job parses the stored body into one of the provider's typed events and calls the provider's `process`, which writes like a Command.

The sender gets its answer within milliseconds, whatever processing costs, and processing that fails is retried from the stored request, not by the sender.

## A provider

A provider is a struct that subclasses `Punch::Webhook::Inbound`, named by `@[Punch::Webhook::Provider]`. The reference app's:

```crystal source=reference/app/billing/payments/provider.cr select=Billing::Payments::Provider
# app/billing/payments/provider.cr
module Billing::Payments
  # ...

  # The payment provider. Its secret is the optional credential
  # `payments.webhook_secret` (and `previous_webhook_secret` during a
  # rotation); without one the endpoint answers 404.
  @[Punch::Webhook::Provider("payments", body_limit: 64 * 1024)]
  struct Provider < Punch::Webhook::Inbound
    def verifier : Punch::Webhook::Verifier?
      secrets = Invoicing::Credentials.current.payments.webhook_secrets || return
      Punch::Webhook::HMAC.new(secrets)
    end

    # `payment.succeeded` is handled; other events are ignored.
    def parse(request : Punch::RawRequest) : Succeeded | Punch::Webhook::Ignored | Punch::Webhook::Unprocessable
      case envelope = Envelope.from_json(request.body)
      in Punch::Rejection then Punch::Webhook::Unprocessable.new("no event type")
      in Envelope
        return Punch::Webhook::Ignored.new(envelope.event) unless envelope.event == "payment.succeeded"

        case succeeded = Succeeded.from_json(request.body)
        in Punch::Rejection then Punch::Webhook::Unprocessable.new("payment.succeeded without a valid payment")
        in Succeeded        then succeeded
        end
      end
    end

    # Marks the invoice paid. A payment for an invoice that is paid
    # already changes nothing; one for an unknown invoice or of the wrong
    # amount fails, which publishes `Punch::Webhook::ProcessingFailed`.
    def process(event : Succeeded, db : Punch::Writer, events : Punch::Events(InvoicePaid)) : Punch::Webhook::Outcome
      payment = event.data
      id = Invoices::Queries.id_by_number(db, number: payment.invoice_number).try(&.id)

      case invoice = id && Invoices::Store.load(db, id)
      in Nil, Invoices::Invoice::Draft then Punch::Webhook::Failed.new("no sent invoice #{payment.invoice_number}")
      in Invoices::Invoice::Paid       then Punch::Webhook::Processed.new
      in Invoices::Invoice::Sent
        case paid = invoice.pay(payment.amount, payment.currency, payment.paid_at, payment.payment_id)
        in Invoices::Invoice::WrongAmount
          Punch::Webhook::Failed.new("#{payment.amount} #{payment.currency} does not settle #{payment.invoice_number} (#{paid.total} #{paid.currency})")
        in Invoices::Invoice::Paid
          Invoices::Store.save(db, paid)
          events.publish(InvoicePaid.new(paid.id, paid.customer_id))
          Punch::Webhook::Processed.new
        end
      end
    end
  end
end
```

Three methods make a provider:

- **`verifier`** returns how deliveries are signed, here `Punch::Webhook::HMAC` with the secrets from the app's credentials. Without a secret it returns `nil`, and the endpoint answers 404: an app that has no secret takes no payments.
- **`parse`** turns the raw body into a typed event, here `Succeeded`, or says it is an event the app does not handle (`Ignored`) or one it cannot read (`Unprocessable`). It parses with the same `Punch::Input` types as an operation's input.
- **`process`** handles one event type. It receives the event and a `Punch::Writer`, and may ask for `events` and `jobs` like a Command's `call`, so `punch events` lists the provider as the publisher of `InvoicePaid`. It returns an outcome: `Processed`, or `Failed` with a reason.

The name in the annotation, `payments`, is the endpoint, `POST /webhooks/payments`. `punch webhooks` lists every provider:

```console
$ punch webhooks
payments  POST /webhooks/payments  Billing::Payments::Provider  verify: edge
```

## Signatures

`Punch::Webhook::HMAC` verifies the Standard Webhooks scheme: an HMAC-SHA256 over the delivery's ID, timestamp, and body, compared in constant time, with five minutes of tolerance against replays. A provider that signs another way gets a verifier of its own, a `Punch::Webhook::Verifier`.

With the default, `verify: :edge`, a bad, missing, or old signature is refused with 401 `invalid_signature`, and nothing is stored. A provider that must get a 2xx quickly whatever happens can use `verify: :deferred`: the request is stored first and checked in the job, against the time it arrived. A provider with `verify: :none` checks nothing; `punch webhooks` and `punch check` list it, so it is never an accident.

**Rotating a secret.** The verifier takes several secrets, and a delivery signed with any of them passes. When the provider rotates, put the new secret beside the old one in the credentials, then remove the old one once the provider signs only with the new. The reference app's credentials hold `webhook_secret` and `previous_webhook_secret` for this ([Credentials](../credentials.md)).

## Duplicates

Senders retry, so a delivery can arrive twice. Each one has a key: the Standard Webhooks `webhook-id`, or whatever `idempotency_key(request)` returns, or else the body's hash. A unique index on the provider and the key decides:

| Delivery | Answer |
|---|---|
| new | 204; stored, and processing enqueued |
| same key, same body | 204; nothing stored, nothing processed again |
| same key, other body | 409 `idempotency_key_reused` |
| bad signature | 401 `invalid_signature`; nothing stored |
| no secret configured | 404 |

Processing is safe to repeat too: the job runs at least once, and an invocation that is already finished is left alone.

## When processing fails

Each stored delivery is an invocation that moves through states. Only a verified one reaches `parse` and `process`, so unverified data is never processed:

| `parse` or `process` returns | The invocation |
|---|---|
| `Processed` | `processed`, in the same transaction as `process`'s writes |
| `Ignored`, from `parse` | `ignored`, with the reason |
| `retry(reason)` | stays verified; `process`'s writes roll back, and the job is retried |
| `discard(reason)` | `discarded`, with the reason; `process`'s writes roll back; not retried, and nobody is told |
| `Failed` | `failed`, with the reason; not retried |
| `Unprocessable` | `unprocessable`; not retried |
| an exception | stays verified, recorded as a bug, and retried |

Return `retry` for what may pass, such as a payment that arrives before its invoice is sent; `discard` for what is nothing to do, such as a test payment from the provider's dashboard; and `Failed` for what will not pass and someone should look at, such as a payment of the wrong amount. A delivery that ends `failed` or `unprocessable` publishes `Punch::Webhook::ProcessingFailed`; subscribe to it to tell someone.

## Inspecting and retrying

The ops console lists invocations at `/_punch/webhooks` by provider and state, shows each stored request, and retries a failed or rejected one, or replays a processed one, through the outbox ([Ops data](../ops.md)). `punch webhooks` counts invocations per state.

Stored bodies often hold personal data. Processed, ignored, discarded, and rejected invocations are purged after 30 days (`PUNCH_WEBHOOK_RETENTION_DAYS`); failed and unprocessable ones stay until someone has looked at them.

The invocations live in Punch's table `punch_webhooks`, in the app's migrations. Write it with `punch db:migration --punch webhooks` ([Migrations](migrations.md#punchs-own-tables)).

## Testing webhooks

`deliver_webhook` posts a body signed by the Standard Webhooks scheme, and `perform_jobs` processes it. The reference app's spec of a payment:

```crystal source=reference/spec/billing/payments/provider_spec.cr select="the payment webhook stores the delivery, then marks the invoice paid once and forgets the customer's statement"
# spec/billing/payments/provider_spec.cr
# ...

describe "the payment webhook" do
  it "stores the delivery, then marks the invoice paid once and forgets the customer's statement" do
    client = AccountsSpec.client
    id, number = sent_invoice(client, unique_email)
    Punch::Spec.perform_jobs # the invoice mail
    body = payment(number)

    -> { deliver(body, id: "msg_once").status.should eq(HTTP::Status::NO_CONTENT) }.should publish(Billing::InvoicePaid, times: 0)
    client.get(Billing::Invoices::Show.path(invoice_id: id)).json["status"].should eq("sent")
    -> { Punch::Spec.perform_jobs.map(&.type).should eq(["Punch::Webhook::Process"]) }.should publish(Billing::InvoicePaid, invoice_id: id, times: 1)
    shown = client.get(Billing::Invoices::Show.path(invoice_id: id)).json
    shown["status"].should eq("paid")
    shown["paid_at"].should eq("2026-10-12T09:30:00Z")
    shown["overdue"].should be_false
    last_invocation.should be_a(Punch::Webhook::Invocation::Processed)

    # The same delivery again: answered, nothing stored or processed.
    deliver(body, id: "msg_once").status.should eq(HTTP::Status::NO_CONTENT)
    Punch::Spec.perform_jobs.should be_empty
    Punch::Spec.webhooks("payments").size.should eq(1)
    Punch::Spec.db.read { |db| db.query_all("SELECT count(*) FROM invoice_payments", &.read(Int64)) }.should eq([1])
    # Another payload under the same delivery ID: 409.
    deliver(payment(number, amount: 1), id: "msg_once").status.should eq(HTTP::Status::CONFLICT)

    statement = client.get(Billing::Customers::Statement.path(customer_id: InvoicingSpec.parse(Billing::Customers::ID, shown["customer_id"].as_i64)))
    statement.json["balances"][0]["outstanding"].should eq(0)
    statement.json["balances"][0]["total"].should eq(25_000)
  end

  # ...
end
```

And of a forged one:

```crystal source=reference/spec/billing/payments/provider_spec.cr select="the payment webhook refuses a delivery without a valid signature, and stores nothing"
# spec/billing/payments/provider_spec.cr
# ...

describe "the payment webhook" do
  # ...

  it "refuses a delivery without a valid signature, and stores nothing" do
    client = AccountsSpec.client
    id, number = sent_invoice(client, unique_email)
    body = payment(number)
    wrong_secret = "whsec_#{Base64.strict_encode("someone-else-entirely-0123456")}"

    deliver(body, secret: wrong_secret).json["code"].should eq("invalid_signature")
    deliver(body, at: Time.utc - 10.minutes).status.should eq(HTTP::Status::UNAUTHORIZED)
    unsigned = Punch::Spec.client.post(Billing::Payments::Provider::Receive.path, json: JSON.parse(body))
    unsigned.status.should eq(HTTP::Status::UNAUTHORIZED)
    Punch::Spec.webhooks.should be_empty
    client.get(Billing::Invoices::Show.path(invoice_id: id)).json["status"].should eq("sent")
  end

  # ...
end
```

`Punch::Spec.webhooks` returns the example's invocations in their states; `id:` and `at:` on `deliver_webhook` set the delivery's key and signing time ([Inbound webhooks: specs](../webhooks.md#specs)).

## Common mistakes

**A Command of your own for deliveries.** Signature checks, duplicates, storage, and retries would all be yours to get right. Write a provider.

**Processing in the request.** A provider that wants to answer only after processing gets retries from the sender and duplicate work. Receive first; process in the job.

**`verify: :none` for a provider that signs.** Anyone could post a payment. Configure its secret and a verifier.

**Calling another system from `process`.** `process` runs in a write transaction. Enqueue a job from it with `jobs : Punch::Jobs(…)`.

**Raising for a bad payload.** An exception is retried as a bug. Return `Unprocessable` from `parse`, or `Failed` from `process`.

## Limits

- Deliveries are stored whole; bodies are limited to 256 KiB unless the provider sets `body_limit:`.
- Signatures other than Standard Webhooks need a verifier of your own.
- Outbound webhooks, the app notifying others, are a deferred subscriber's job; Punch has no outbound delivery of its own.

## Rules

- Accept another system's deliveries only through a provider, never a Command of your own. ([A provider](#a-provider))
- Verify every provider that signs; `verify: :none` only for one that does not. ([Signatures](#signatures))
- Return `Ignored`, `Unprocessable`, `Failed`, `retry(reason)`, or `discard(reason)` for expected outcomes; raise only for bugs. ([When processing fails](#when-processing-fails))
- Never call another system inside `process`; enqueue a job. ([Common mistakes](#common-mistakes))
- Rotate a secret by listing the new one beside the old one first. ([Signatures](#signatures))

## Reference

- [Inbound webhooks](../webhooks.md): every option, the endpoint's answers, verification, states, storage, telemetry, and spec helpers.
- [Background jobs](../jobs.md): how processing is retried.
