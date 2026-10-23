# Tracing and Metrics

Applies to Punch 0.1.

When something is slow or wrong in production, the first question is what one request did. Punch answers it without an agent or a third-party service: every request and every job run is a trace, whose root span is one wide event with everything about it, from the route and the outcome to how many queries it ran and how long it waited for the writer. Metrics add how the process is doing between requests. Both are on by default, and both follow OpenTelemetry's model, so they can be exported to any collector.

After reading this guide, you will know:

- what a request and a job run record, and what is never recorded;
- how to see a trace in development;
- how to add your own fields and spans;
- how to export traces and metrics to an OpenTelemetry collector;
- which metrics to watch.

Before you start: you know Queries, Commands, and jobs ([Operations and Routing](operations-and-routing.md), [Background Jobs](background-jobs.md)).

## One wide event per request

Every request becomes one JSON line on standard output, its wide event. Two requests to the examples app, an API call without a session and a page:

```console
$ bin/examples serve
{"time":"2026-10-23T15:25:15.184602Z","name":"GET /accounts/users/me","trace_id":"ec9c5a352f5de4a681c5013c9d6dcd2e","span_id":"26c68a3d83856d0d","duration_ms":20.368,"http.request.method":"GET","url.path":"/accounts/users/me","url.scheme":"http","client.address":"127.0.0.1","user_agent.original":"curl/8.18.0","punch.request_id":"ec9c5a352f5de4a681c5013c9d6dcd2e","http.route":"/accounts/users/me","punch.operation":"Accounts::Users::Me","punch.surface":"json","punch.outcome":"unauthenticated","punch.locale":"en","http.response.status_code":401,"http.response.body.size":158}
{"time":"2026-10-23T15:25:15.208387Z","name":"GET /accounts/sessions/new","trace_id":"850e13f5b48b8d22b71b2d8caa061012","span_id":"485207a2f9dc5643","duration_ms":1.494,"http.request.method":"GET","url.path":"/accounts/sessions/new","url.scheme":"http","client.address":"127.0.0.1","user_agent.original":"curl/8.18.0","punch.request_id":"850e13f5b48b8d22b71b2d8caa061012","http.route":"/accounts/sessions/new","punch.operation":"Accounts::Sessions::CreatePage","punch.surface":"html","punch.outcome":"sign_in_view","punch.locale":"en","http.response.status_code":200,"http.response.body.size":1280}
```

The fields follow OpenTelemetry's names where one exists, and Punch's own start with `punch.`:

- **What was asked:** method, path, route, the operation that answered, the surface (JSON, HTML, markdown).
- **How it ended:** the status and `punch.outcome`, the result type's name in snake_case, such as `sent` or `not_found`, or `invalid_input` for a rejected input.
- **Who:** `user.id` when someone is signed in.
- **What it cost:** the duration, and totals from every statement: `punch.db.queries`, `punch.db.rows`, `punch.db.duration_ms`, and `punch.db.writer_wait_ms`, how long the request waited for the database's writer.
- **What went wrong:** on a failure, `error`, and with an exception its type and message.

`punch.request_id` is also in every error response's body, so a user's report leads straight to the line. A job run is a wide event of its own, continuing the trace of the request that enqueued it.

One line with every field beats many log lines with a few each: it can be filtered and grouped by any field, and a slow request carries its own explanation.

**Never recorded:** SQL arguments, request and response bodies, query strings, and headers other than `User-Agent`. A `Punch::Secret` cannot be put on a span; that does not compile.

## Seeing a trace in development

`punch dev` prints each request's trace as a tree, with a span for the input, each transaction, and each SQL statement, named after its named query. Set `PUNCH_TRACE=console` for the same with `serve`. In specs, `response.trace` is the request's trace, and `should have_span(...)` checks it.

## Adding your own fields

Business fields go on the wide event, with the app's own prefix. The examples app's reminder records which invoice it was about:

```crystal source=docs/guides/examples/app/billing/invoices/jobs/remind.cr select=Billing::Invoices::Remind#perform
# app/billing/invoices/jobs/remind.cr
module Billing::Invoices
  # Mails the customer of one overdue invoice a reminder, and records
  # when. Safe to run again: an invoice that was paid, or reminded in the
  # meantime, is skipped. At most two run at a time, so the reminders of
  # a big sweep leave the mail relay's other connections to sign-in mail.
  @[Punch::Job::Options(queue: "mail", attempts: 5)]
  @[Punch::Job::Concurrency(2)]
  @[Punch::Payloads::Version(2, from: {1 => V1})]
  struct Remind
    # ...

    def perform(job : Punch::Job::Context) : Punch::Job::Result
      # On the run's wide event, so a reminder can be found by its invoice.
      Punch::Trace.event["invoice.id"] = invoice_id.value
      Punch::Trace.event["invoice.reminder"] = reminder.to_s.underscore
      invoice = job.read { |db| Table.find(db, invoice_id) }
      return discard("invoice #{invoice_id} is gone") unless invoice
      return discard("invoice #{invoice_id} is paid") if invoice.paid_at
      if reminded = invoice.reminded_at
        return discard("invoice #{invoice_id} was reminded already") if reminded > job.now - REMINDER_INTERVAL
      end

      result = Punch::Mailer.deliver(ReminderMail.new(invoice, reminder), locale: Punch::I18n.locale(nil), job: job)
      job.write { |db| Queries.mark_reminded(db, id: invoice_id, now: job.now) } if result.nil?
      result
    end

    # ...
  end
end
```

`Punch::Trace.event` is the wide event of the current request or job, wherever the code runs in it. Values are strings, booleans, numbers, or arrays of strings; convert domain types where you set them. Now every reminder's run can be found by its invoice. A spec checks the field through the ops store, which records job runs as `serve` does:

```crystal source=docs/guides/examples/spec/billing/invoices/jobs/remind_spec.cr select="Billing::Invoices::Remind records the invoice on the run's wide event"
# spec/billing/invoices/jobs/remind_spec.cr
describe Billing::Invoices::Remind do
  # ...

  it "records the invoice on the run's wide event" do
    ops = Punch::Spec.ops
    invoice = BillingSpec.invoice(due_at: Punch::Spec.job_clock.now - 3.days)
    Punch::Job::Queue.enqueue(Billing::Invoices::Remind.new(invoice.id, :final))
    Punch::Spec.perform_jobs
    run = ops.records(kind: "job").find!(&.name.ends_with?("Billing::Invoices::Remind"))
    fields = JSON.parse(run.fields)
    {fields["invoice.id"], fields["invoice.reminder"]}.should eq({invoice.id.value, "final"})
  end

  # ...
end
```

A span of your own measures one piece of work, such as a call to another system in a job:

```crystal illustrative="a child span around an outbound call"
Punch::Trace.span("charge card", :client) do |span|
  span["payment.provider"] = "stripe"
  gateway.charge(amount)
end
```

It records an exception, finishes the span, and raises again. An exception your code handles, but that someone should see, goes on the trace with `Punch::Trace.event.record_exception(ex, "where")`; the ops console lists it.

## Exporting to OpenTelemetry

Set OpenTelemetry's own variables, and `serve` and `jobs` send traces and metrics to a collector or a vendor's endpoint, over OTLP with HTTP and JSON:

```sh illustrative="the settings of an export; the values are your collector's"
OTEL_EXPORTER_OTLP_ENDPOINT=https://otlp.example.com
OTEL_EXPORTER_OTLP_HEADERS="x-api-key=…"
OTEL_RESOURCE_ATTRIBUTES="service.version=1.4.0"
PUNCH_TRACE_SAMPLE_PERCENT=10
```

- Every span of each sampled trace is sent, with its attributes and exceptions. `PUNCH_TRACE_SAMPLE_PERCENT` decides the share of traces, by trace ID; wide events still go to standard output for every request.
- Export never holds up a request: finished traces are queued and sent in batches, and dropped when the queue is full.
- `bin/<app> config` shows the endpoint, the header names, and the service name, never the headers' values.

Recording spans costs throughput: about 10 % at saturation with every trace sampled. Under heavy load, sample ([Tracing: cost](../tracing.md#cost)).

## Metrics

Metrics say how the process is doing between requests. Punch's own are the signals of each database:

| Metric | Watch for |
|---|---|
| `db.client.connection.pending_requests` | writes waiting for the writer; above a few for long means long or slow write transactions |
| `punch.db.wal.size`, `punch.db.wal.frames` | a WAL that keeps growing: a long read keeps it from being checkpointed |
| `punch.db.checkpoint.duration` | checkpoints of tens of milliseconds stall every write behind them |
| `punch.db.size` | the database growing towards the disk's size |
| `punch.backup.lag`, `punch.backup.verification.age` | a replica falling behind, a restore check that stopped passing |

They are exported with the traces when `OTEL_EXPORTER_OTLP_ENDPOINT` is set, every minute by default. An app can add its own instruments to the same registry ([Metrics](../metrics.md#collecting)).

## Common mistakes

**Logging instead of adding fields.** A log line per step scatters a request over many lines. Set fields on `Punch::Trace.event`.

**Putting personal data in fields.** A wide event is kept and exported. Record IDs, not names, addresses, or message bodies.

**Unprefixed field names.** `id` collides with someone else's `id`. Prefix with the app's own namespace: `invoice.id`.

**Exporting every span under heavy load.** It costs throughput. Set `PUNCH_TRACE_SAMPLE_PERCENT`.

## Limits

- Wide events go to standard output as JSON lines; collecting and keeping them is the deployment's job, or the ops console's for errors, job runs, and slow requests.
- OTLP is HTTP with JSON only, not protobuf or gRPC.

## Rules

- Find what a request did by its wide event and its `punch.request_id`. ([One wide event per request](#one-wide-event-per-request))
- Add business fields to `Punch::Trace.event`, prefixed with the app's namespace, with IDs rather than personal data. ([Adding your own fields](#adding-your-own-fields))
- Export with OpenTelemetry's own `OTEL_*` variables, and sample under heavy load. ([Exporting to OpenTelemetry](#exporting-to-opentelemetry))
- Watch the writer's queue, the WAL, and the backup's lag. ([Metrics](#metrics))

## Reference

- [Tracing and wide events](../tracing.md): every field, tracers and sinks, the export's settings, and the cost.
- [Metrics](../metrics.md): every metric, collecting, and the app's own instruments.
- [Ops data](../ops.md): what the ops console keeps from the wide events.
