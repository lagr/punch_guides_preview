# Building a JSON API

Applies to Punch 0.1.

Every Query and Command in a Punch app is already a JSON endpoint. There are no serializers to register and no API controllers to write: the operation's input is the request, its `Output` is the set of responses, and the OpenAPI document of all of them is derived from those types and committed as `openapi.json`. This guide shows what a client sees, how the document is produced and kept honest, what counts as a breaking change, and how to make Commands safe to retry.

After reading this guide, you will know:

- what a client sends and receives for each kind of outcome;
- how `openapi.json` is produced, why it is committed, and how specs check every response against it;
- what counts as a breaking change, and how `punch check` catches one;
- how to make a Command safe to retry with an `Idempotency-Key`;
- how to answer repeated reads with `304 Not Modified`;
- how a program authenticates, from `curl` and from a generated client.

Before you start: you know [Results, Errors, and Status Codes](results-errors-and-status-codes.md). The examples come from the reference app, an invoicing app in `reference/`, run on a laptop with an API token.

## Calling an operation

A program calls the reference app with an API token in the `Authorization` header. Commands take JSON bodies; Queries take query strings:

```console
$ curl -X POST http://localhost:3000/billing/invoices \
    -H "Authorization: Bearer inv_…" -H "Content-Type: application/json" \
    -d '{"customer_id":1,"currency":"EUR","lines":[{"description":"Support","quantity":1,"unit_price":9000}]}'
{"invoice":{"id":2,"issuer":{"name":"Example Ltd"},"status":"draft","customer_id":1,"currency":"EUR","lines":[…],"total":9000,"created_at":"2026-10-23T19:44:54Z","overdue":false}}

$ curl http://localhost:3000/billing/invoices/2 -H "Authorization: Bearer inv_…"
{"id":2,"issuer":{"name":"Example Ltd"},"status":"draft",…}
```

The route comes from the operation's name, and the shape of each answer from its result type. `Invoices::Create` returns `Created`, with status 201 and the invoice under `invoice`; `Invoices::Show` returns `Detail`. `punch routes` lists every endpoint with its method, path, and policy ([Operations and Routing](operations-and-routing.md#checking-the-routes)).

A result is a struct with `JSON::Serializable`, written for the client. A generated database row has no JSON form, and returning one does not compile (`Billing::Customers::Record is a database row, not a response`). That keeps the wire shape a declared choice: adding a column never leaks it into the API.

## What a failure looks like

Every failure has one shape, Problem Details (RFC 9457), with a `code` a client branches on and a `request_id` that leads to the request's trace. A refused input lists every error at its field:

```console
$ curl -i -X POST http://localhost:3000/billing/invoices -H "Authorization: Bearer inv_…" \
    -H "Content-Type: application/json" \
    -d '{"customer_id":1,"currency":"EURO","lines":[{"description":"","quantity":0,"unit_price":100}]}'
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{"type":"about:blank","title":"Unprocessable Entity","status":422,"code":"invalid_input","request_id":"aeb0c6f00f355291437966cef6460940","detail":"Some of the input is not valid.","errors":[{"path":["currency"],"code":"invalid_currency","params":{},"message":"Is not a currency code."},{"path":["lines",0,"description"],"code":"blank","params":{},"message":"Must not be empty."},{"path":["lines",0,"quantity"],"code":"out_of_range","params":{"min":1,"max":10000},"message":"Is out of range."}]}
```

A failure the operation returns carries its own fields:

```console
$ curl -i -X POST http://localhost:3000/billing/invoices/2/send -H "Authorization: Bearer inv_…" -H "Idempotency-Key: 9b77e1"
HTTP/1.1 409 Conflict

{"type":"about:blank","title":"Conflict","status":409,"code":"already_sent","request_id":"0772bc9160bcf45a2bfac515706616e0","detail":"This invoice was already sent.","number":"INV-000002","sent_at":"2026-10-23T19:44:54Z"}
```

And a request without credentials gets the challenge:

```console
$ curl -i http://localhost:3000/billing/invoices/1
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer

{"type":"about:blank","title":"Unauthorized","status":401,"code":"unauthenticated","request_id":"7d4f1a4d4fcf79d7036cdb2dae0d1714","detail":"Please sign in."}
```

`code` is stable; `detail` and `message` are for people, rendered from the locale files in the language the request asked for (`Accept-Language`, or the user's stored language). Clients must branch on `code`, never on text. Where codes and messages come from is in [Results, Errors, and Status Codes](results-errors-and-status-codes.md#problem-details).

## The OpenAPI document

`punch openapi` writes `openapi.json`, an OpenAPI 3.2 document of every operation. It is derived, never written by hand:

| Part of the document | Comes from |
|---|---|
| Paths and methods | the route table |
| `operationId`, tag | the operation's name (`billing.invoices.send`) and its slice (`billing`) |
| `summary`, descriptions | doc comments on the operation, its types, and their getters |
| Parameters, request body | the input's fields and their types, with every constraint a domain primitive declares |
| Responses | each member of `Output`, by its status; problems with their `code` |
| 401, 403, 404, 422, 429 | what the operation's actor, policy, input, and rate limits make possible |

Here is part of the entry for sending an invoice:

```json illustrative="an excerpt of reference/openapi.json, shortened"
{
  "operationId": "billing.invoices.send",
  "summary": "Sends a draft: it gets the next invoice number and is due after the payment term.",
  "tags": ["billing"],
  "parameters": [
    {"name": "invoice_id", "in": "path", "required": true,
     "schema": {"type": "integer", "format": "int64", "minimum": 1}},
    {"name": "Idempotency-Key", "in": "header", "required": true, "description": "Names this attempt, so it can be retried safely: …"}
  ]
}
```

with responses `200`, `400`, `401`, `403`, `404`, `409`, `422`, `429`, and `default`. The responses come from **the declared `Output`**, not from what `call` happens to return today. If `Send` declares `NoLines` but no code path returns it yet, the document still promises it. The declaration is the contract.

> [!IMPORTANT]
> Write the doc comment of every operation and result type for the client. Its first sentence becomes the operation's `summary` in `openapi.json`, and every getter's comment becomes its property's description.

The app serves the committed file at `GET /openapi.json`, embedded in the binary at compile time.

## Why the document is committed

`openapi.json` is in the repository, like `db/schema.sql`. Three things follow:

1. **A reviewer sees the API change.** A pull request that adds a required field shows it in the diff of `openapi.json`, next to the code that caused it.
2. **`punch check` fails when it is stale.** Its `openapi` step regenerates the document and compares. When you change an operation, run `punch openapi` and commit the result. `punch dev` does not regenerate it on every save, because that needs a second build.
3. **Specs check every response against it.** In an endpoint spec, each response the client receives is compared with the document for its operation: the status must be documented, the content type named, and the body must match the schema. A spec fails at the line of the request:

   ```text
   GET /billing/invoices/7 answered 200, which breaks the OpenAPI document:
     body (root): "due_on" is missing (required)
   If the code is right, the document is stale: run 'punch openapi'.
   ```

   This is a contract test for every operation, for free. It cannot be turned off, and that is deliberate: a document that drifts from the code is worse than none.

## Breaking changes

A change is breaking when a client that was correct against the old document fails against the new one:

- an operation or a success response is gone;
- a request needs more: a new required parameter or property, a body that became required, a type or enum value no longer accepted;
- a response promises less: a property removed or no longer always present, a new type, or a new enum value.

Adding operations, optional inputs, response properties, and accepted values is not breaking.

`punch check` compares the new `openapi.json` with the one on the main branch (`--base`, by default `origin/main`) and fails on a breaking change. Here, removing `overdue` from `Detail`:

```console
$ punch openapi:diff openapi.json tmp/openapi.new.json
punch: 5 breaking changes; raise the major version in shard.yml if intended:
  POST /billing/invoices: response 201.invoice: property overdue was removed
  GET /billing/invoices/{invoice_id}: response 200: property overdue was removed
  PATCH /billing/invoices/{invoice_id}: response 200: property overdue was removed
  POST /billing/invoices/{invoice_id}/send: response 200: property overdue was removed
  GET /billing/invoices/{token}/shared: response 200: property overdue was removed
```

One removed field broke five operations, because they share `Detail`. A diff like this is the moment to decide: rethink the change (keep the field, add a new one beside it), or accept the break and raise the version in `shard.yml`, which becomes `info.version` in the document. Before 1.0, raising the minor version is enough. The reference app went from 0.1.0 to 0.2.0 when its invoice status gained `paid`, and to 0.3.0 when sending started to require an idempotency key.

> [!NOTE]
> **Coming from Rails:** where a Rails app might describe its API with rswag specs or a hand-kept OpenAPI file, a Punch app derives the document from the operations' types. A change to an operation changes `openapi.json` in the same commit. See [Coming from Rails](coming-from-rails.md).

## Safe retries with idempotency keys

A client whose request timed out does not know whether the Command ran. For sending an invoice, guessing wrong mails the customer twice. With an `Idempotency-Key`, the client names the attempt, and a retry with the same key gets the first answer back instead of running the Command again:

```console
$ curl -i -X POST http://localhost:3000/billing/invoices/1/send \
    -H "Authorization: Bearer inv_…" -H "Idempotency-Key: 6f1c2a"
HTTP/1.1 200 OK
Idempotent-Replayed: true

{"id":1,"status":"sent","number":"INV-000001",…}
```

That was the second request with the key: the answer is the first one's, marked `Idempotent-Replayed: true`, and nothing ran. Every Command that takes the app's actor accepts keys, scoped per user and operation. The answer is stored inside the Command's transaction, so a crash leaves either the writes, the outbox entries, and the stored answer, or none of them.

Annotate a Command whose duplicate would hurt, such as one that moves money or sends something, to require a key:

```crystal illustrative="the annotation alone; Billing::Invoices::Send in the reference app carries it"
@[Punch::Idempotency(:required)]
struct Send < Punch::Command
```

The other answers a client can get:

| Situation | Answer |
|---|---|
| No key where one is required | 400 `idempotency_key_required` |
| Same key, a different request | 422 `idempotency_key_reused` |
| A key that does not parse | 400 `invalid_idempotency_key` |
| A key where the Command takes none (sign-in, showing a secret once) | 400 `idempotency_key_not_accepted` |

```console
$ curl -i -X POST http://localhost:3000/billing/invoices/2/send -H "Authorization: Bearer inv_…" -H "Idempotency-Key: 6f1c2a"
HTTP/1.1 422 Unprocessable Entity

{"type":"about:blank","title":"Unprocessable Entity","status":422,"code":"idempotency_key_reused","request_id":"c9d8ead12a28703395bb1511f775d790","detail":"This Idempotency-Key was used for a different request."}
```

Clients should make a fresh random key (a UUID) per attempt and reuse it only to retry that same request. Answers are replayed for at least 24 hours (`PUNCH_IDEMPOTENCY_RETENTION_HOURS`), which the document promises. The table that stores them is part of the app's migrations: `punch db:migration --punch idempotency` writes it.

A spec checks a Command's retry behaviour with `client.twice`:

```crystal source=reference/spec/billing/invoices/send_spec.cr select="invoices API sends once under one idempotency key and gives a retry the same answer, and refuses a send without a key"
# spec/billing/invoices/send_spec.cr
# ...

describe "invoices API" do
  # ...

  it "sends once under one idempotency key and gives a retry the same answer, and refuses a send without a key" do
    client = AccountsSpec.client
    id = draft(client, customer_id(client))
    missing = client.post(Billing::Invoices::Send.path(invoice_id: id))
    missing.status.should eq(HTTP::Status::BAD_REQUEST)
    missing.json["code"].should eq("idempotency_key_required")

    send = -> { client.twice("POST", Billing::Invoices::Send.path(invoice_id: id)) }
    send.should enqueue(Billing::Invoices::Remind, invoice_id: id, times: 1)
    Punch::Spec.perform_jobs.size.should eq(1)
    Punch::Spec.mails.size.should eq(1)
    # A new key is a new attempt: the invoice is sent already.
    client.post(Billing::Invoices::Send.path(invoice_id: id), headers: Punch::Spec.idempotency_key).json["code"].should eq("already_sent")
  end

  # ...
end

# ...
```

The guarantee covers what the Command does in its transaction and through the outbox. A Command that calls another system from `call` is outside it, which is one more reason such calls belong in jobs ([Background Jobs](background-jobs.md)).

## Answering repeated reads with 304

A client that polls a Query can be told "nothing changed" without the body. A result type that includes `Punch::HTTP::Versioned` gets an `ETag`, and a request whose `If-None-Match` names the current one gets `304 Not Modified`. The reference app's customer statement does this:

```crystal source=reference/app/billing/customers/operations/statement.cr select=Billing::Customers::Balances
# app/billing/customers/operations/statement.cr
module Billing::Customers
  # ...

  # A customer's balances per currency. Computing it adds up every line of
  # every sent invoice, so it is cached (`STATEMENTS`) and may be up to a
  # minute old; `as_of` says when it was computed. Repeated requests get
  # `304 Not Modified` while the balances are unchanged.
  @[Punch::Status(200)]
  struct Balances
    include JSON::Serializable
    include Punch::HTTP::Versioned

    getter customer_id : ID
    getter balances : Array(Balance)
    getter as_of : Time

    def initialize(@customer_id : ID, @balances : Array(Balance), @as_of : Time)
    end

    # The balances themselves: a recomputed statement with the same
    # numbers keeps its version.
    def version : String
      Digest::SHA256.hexdigest(balances.to_json)
    end
  end

  # ...
end
```

```console
$ curl -i http://localhost:3000/billing/customers/1/statement -H "Authorization: Bearer inv_…"
HTTP/1.1 200 OK
ETag: W/"M0IexfiyEN1yI29emYF9og"
Cache-Control: private, no-cache

{"customer_id":1,"balances":[{"currency":"EUR","invoices":1,"overdue":0,"total":30000,"outstanding":30000}],"as_of":"2026-10-23T19:44:44Z"}

$ curl -i http://localhost:3000/billing/customers/1/statement -H "Authorization: Bearer inv_…" \
    -H 'If-None-Match: W/"M0IexfiyEN1yI29emYF9og"'
HTTP/1.1 304 Not Modified
ETag: W/"M0IexfiyEN1yI29emYF9og"
```

`version` returns whatever changes when the content does: an `updated_at`, a revision, a digest. The tag also covers the user and the running binary, so users never share one and a deploy changes all of them. The result is still computed before the tag is compared: a 304 saves the transfer, not the server's work. Cache the result as well when computing it is what costs, as the statement does ([Caching](caching.md#conditional-requests)).

## Tokens for programs

A program authenticates with an API token in `Authorization: Bearer …`. A token acts as its user, with that user's roles, always expires, and is shown once when it is made. In development, make one on the command line:

```console
$ bin/invoicing tokens:create --user 1 --name dev
inv_rYW3udepRFou6wnvZ6JptaP5NfltJgXjWSE6WWImmEO2Bqbrm
Token 1 for user 1, expires 2026-11-22 19:40. It is shown only now.
```

In production, users make their own on a page of the app; `punch new` ships it as `app/accounts/tokens/`. Making a token needs a recent sign-in with a password, so a token can never make another token. The prefix (`inv_` in the reference app, set in `app/setup.cr`) lets people and secret scanners tell whose token it is, and a checksum lets the app refuse a mistyped one without a lookup. See [Authentication](authentication.md#sessions-and-api-tokens).

Requests with a token are not subject to the cross-site check that guards browser sessions, because a browser never attaches one by itself.

## Generating a client

Because `openapi.json` describes every request and response, a client can be generated from it with any OpenAPI generator, for example:

```console illustrative="a third-party generator; Punch does not ship one"
$ npx @openapitools/openapi-generator-cli generate -i openapi.json -g typescript-fetch -o client/
```

Each operation becomes a method named after its `operationId`, each result type a model, and each problem type an error with its `code`. Regenerate the client when `openapi.json` changes, ideally in the same pull request. Check that the generator you choose reads OpenAPI 3.2, the version Punch writes.

## Common mistakes

**Returning a record from a Query.** `call` returns a generated `Record`, and the build fails:

```crystal fails=spec/fail/record_as_response.cr
# A generated `Record` is a database row, not a wire type: returning one in
# a result does not compile, and says what to return instead.
module Customers
  struct Found
    include JSON::Serializable
    getter customer : Ledger::Customers::Record

    def initialize(@customer : Ledger::Customers::Record)
    end
  end

  @[Punch::Public]
  struct Index < Punch::Query
    struct Input
      include Punch::Input
    end

    def call(input : Input, db : Punch::Reader) : Found
      raise "never run"
    end
  end
end

Punch::Routing::Router.new(Punch::Database.memory)
Punch::Routing.render(Customers::Found.allocate, Punch::HTTP::Context.allocate)
```

```text
Error: Ledger::Customers::Record is a database row, not a response: return a result struct with the fields to show
```

Return a result struct with the fields the client should see, built from the row. That keeps a new column from becoming part of the API by accident.

**Committing code without `openapi.json`.** `punch check` fails its `openapi` step with `stale_openapi`. Run `punch openapi` and commit the document with the code.

**Silencing a breaking change.** Removing a property and regenerating the document turns `punch check` red against the main branch. Do not work around the diff; either keep the old field beside the new one for a release, or raise the version and tell your clients.

**Branching on `detail` or `message`.** They are translated text and change with the language and the wording. Branch on `code`.

**Retrying with a new key.** A client that makes a new key for each retry defeats the replay: the second attempt is a new request, and gets `already_sent` or, for a Command without such a check, runs twice. Keep the key until you have an answer.

**Calling an outside system from `call` and relying on the key.** The key covers the transaction, not a payment provider called in the middle. Enqueue a job for the outside call.

**Leaving an outcome out of `Output`.** A `call` that returns a type its `Output` does not list does not compile, so the code cannot answer more than the document says. The other direction is allowed on purpose: declare an outcome a client must handle even before the code produces it, since the document follows the declaration.

## Limits

- JSON only. There is no XML, MessagePack, or form-encoded API surface; forms post to Commands from pages only.
- No API versioning by URL or header. One document describes the current API; breaking changes are versioned in `info.version`.
- No OAuth or scoped tokens. A token acts as its user, with all of that user's permissions.
- Multipart bodies and uploads are refused.
- `304` saves the transfer, not the computation.

## Rules

- Return result structs written for the client, never database rows. ([Calling an operation](#calling-an-operation))
- Declare every outcome in `Output`; the document follows the declaration. ([The OpenAPI document](#the-openapi-document))
- Commit `openapi.json` with the code that changed it. ([Why the document is committed](#why-the-document-is-committed))
- Never turn off the contract check; when a response breaks the document, fix the code or regenerate the document. ([Why the document is committed](#why-the-document-is-committed))
- Treat a breaking change as a decision: avoid it, or raise the version. ([Breaking changes](#breaking-changes))
- Require an idempotency key on Commands that move money or send something. ([Safe retries with idempotency keys](#safe-retries-with-idempotency-keys))
- Clients branch on `code`, never on text. ([What a failure looks like](#what-a-failure-looks-like))

## Reference

- [JSON endpoints and OpenAPI](../openapi.md): statuses, Problem Details, every part of the document, `openapi:diff`.
- [JSON Schema](../json-schema.md): how types become schemas.
- [Operations and routing](../operations.md#idempotency-keys): idempotency keys in full.
- [Cache](../cache.md#conditional-requests): conditional requests.
- [Authentication](../auth.md#api-tokens): API tokens and tokens for agents.
- [Testing](../testing.md#the-contract-check): the contract check.
