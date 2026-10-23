# Application Structure

Applies to Punch 0.1.

In a Punch app, a type's full name decides its file. `Billing::Invoices::Send` lives in `app/billing/invoices/operations/send.cr`, answers `POST /billing/invoices/:invoice_id/send`, and is called `billing.invoices.send` in `openapi.json` and in permission checks. So a name you see anywhere (in a trace, a route list, an error, the ops console) leads to one file without a search, and anything new has exactly one place to go. This guide shows that layout, the reasons for it, and the checks that keep it.

After reading this guide, you will know:

- what slices, namespaces, and entities are, and how they map to directories;
- where to put a new operation, page, component, policy, model, query, job, mail, or subscriber;
- what an outline is, and why every namespace has one;
- how files are required without a single `require` line of your own;
- what `punch new` and `punch generate` write for you;
- which check reports a file in the wrong place, and what it says.

Before you start: you have read [How a Punch App Works](how-a-punch-app-works.md). The examples come from the reference app, an invoicing app in `reference/`.

## The shape of an app

An app made by `punch new` looks like this at the top:

```text
app/                 the app's code; everything below is required for you
  setup.cr           every app-wide Punch declaration
  actor.cr           who is asking: the app's actor type
  credentials.cr     what the app needs from its deployment
  languages.cr       the languages the app offers
  components/        layouts and components the whole app shares
  assets/            stylesheets, images, scripts
  accounts.cr        the outline of the accounts slice
  accounts/          the accounts slice: users, sessions, roles, …
  home.cr  home/     the home page
config/
  locales/           the app's text, per language
  credentials/       encrypted credentials per environment
db/
  migrations/        plain SQL, one file per change
  schema.sql         the schema the migrations produce (generated)
  seeds.cr           development data
spec/                specs, mirroring app/
src/
  invoicing.cr       the library: requires Punch and app/requires.gen.cr
  main.cr            the binary: the library plus the command line
openapi.json         the API contract (generated)
payloads.lock.json   job payload types (generated)
AGENTS.md            the guide for people and agents working on this app
```

You write almost everything under `app/`, `db/migrations/`, `config/locales/`, and `spec/`. The rest is set up once.

## Slices, namespaces, and entities

The reference app's billing slice shows the three levels:

```text
app/
  billing.cr                     outline of the slice
  billing/                       a slice
    events.cr                    the slice's domain events
    customers.cr                 outline of the entity
    customers/                   an entity: it owns a table
    invoices.cr                  outline of the entity
    invoices/                    an entity: it owns tables
    payments.cr                  outline of the namespace
    payments/                    a namespace without tables
      provider.cr                the payment provider's webhooks
```

- **A slice** is a directory directly under `app/`: a bounded part of the domain, such as `billing` or `accounts`. Another slice uses its data only through its operations, never its queries, tables, stores, or records. That keeps each slice's tables free to change.
- **A namespace is a directory,** at any depth. `Accounts::Organizations::Members` is `app/accounts/organizations/members/`. A file's outermost module is its directory's namespace.
- **An entity** is a namespace that owns tables. It maps them in its `columns.cr`, and each table has exactly one owner. Everything about an entity is in its directory, plus the outline beside it.
- **A namespace without tables** holds a concept that has no rows of its own: a webhook provider, a set of roles, a use case that writes several entities.

Nest a namespace under another only when its rows belong to one of the other's: its table has a foreign key to the parent as its owner. The route says so too. Members of an organization live at `app/accounts/organizations/members/` and answer `/accounts/organizations/:organization_id/members`. A grouping that is not ownership stays a sibling in the slice.

## Inside an entity

The invoices entity holds everything about invoices:

```text
app/billing/invoices/
  types.cr                       ID, Number, Quantity: the entity's domain types
  columns.cr                     which column holds which domain type
  records.gen.cr                 the generated record and table (punch db:generate)
  queries.sql                    the entity's named SQL
  queries.gen.cr                 typed methods for it (punch db:generate)
  invoice.cr                     the model: Invoice::Draft, Sent, Paid
  store.cr                       loads and saves the model with the queries
  sharing.cr                     signed links to an invoice
  inputs.cr                      LineInput, which several operations take
  results.cr                     Detail, NotFound, AlreadySent: shared results
  policies.cr                    ByRole: who may do what to an invoice
  components.cr                  LinesTable: shared by several pages
  operations/
    index.cr  show.cr  create.cr  update.cr  destroy.cr  send.cr  link.cr  shared.cr
  pages/
    index_page.cr  show_page.cr  create_page.cr  shared_page.cr
  jobs/
    remind.cr                    Billing::Invoices::Remind
  mail/
    invoice_mail.cr              Billing::Invoices::InvoiceMail
  subscribers/
    mail_invoice.cr              Billing::Invoices::MailInvoice
```

The directories `operations/`, `pages/`, `jobs/`, `mail/`, `subscribers/`, and `components/` name a kind of file and add no namespace: `app/billing/invoices/operations/send.cr` holds `Billing::Invoices::Send`, not `Billing::Invoices::Operations::Send`.

## Where each thing goes

| Thing | File |
|---|---|
| A Query or Command | `operations/<name>.cr`, one per file, with the result types only it returns |
| A page | `pages/<name>_page.cr`, one per file, with the components only it renders |
| A job, mail, or subscriber | `jobs/`, `mail/`, `subscribers/`, one per file, named after the type |
| Results, inputs, policies several operations share | `results.cr`, `inputs.cr`, `policies.cr` in the namespace |
| Components several pages share | `components.cr` in the namespace; app-wide ones in `app/components/`, one file each |
| IDs and domain primitives | `types.cr` of the entity that owns them; the slice's `types.cr` holds only what the slice shares |
| Events | `events.cr` in the slice |
| Column types | `columns.cr` of the entity that owns the table |
| Behaviour of a generated record | `record.cr` beside `columns.cr` |
| A model with states, a store, a helper type | a file named after its type: `invoice.cr`, `store.cr`, `sharing.cr` |
| Named SQL | `queries.sql` in the entity's directory |
| Roles | `app/accounts/roles.cr` |
| App-wide settings (`Punch.assets`, `Punch.translations`, `Punch.credentials`, `Punch.migrations`, `Punch::….config =`) | `app/setup.cr`, never in a type's file |

**One file per operation** is the rule that shapes the rest. Listing `operations/` lists the namespace's API, a trace's operation name gives the file, and the file holds everything that operation can answer: its annotations, `Input`, `Output`, `call`, and the result types nobody else returns. The reference app's `send.cr` defines `NoLines` next to `Send`, because only sending can fail that way. `AlreadySent` is returned by `Send` and by `Destroy`, so it lives in `results.cr`.

Type names stay at the namespace's module even when they live in an operation's file: `Billing::Invoices::NoLines`, not `Billing::Invoices::Send::NoLines`. Moving a type from an operation's file to `results.cr` later changes neither code nor `openapi.json`.

## A use case goes where it writes

An operation lives with the entity whose state it changes, and a Query with the entity it reads most. That also turns verbs into resources. Signing in creates a session, so it is `Accounts::Sessions::Create`, at `POST /accounts/sessions`; signing out is `Accounts::Sessions::SignOut`. Sending an invoice changes the invoice, so it is `Billing::Invoices::Send`.

A use case that writes several entities gets a namespace of its own, without tables, beside them. Signing up creates a user, an email identity, and an organization, so it is `Accounts::SignUp::Create` in `app/accounts/sign_up/`. No operation sits directly in a slice.

Naming a namespace is naming that part of the API. Before adding a namespace, read its route aloud.

## Outlines: what a namespace means

Every namespace directory has an outline: the file beside it, with the same name, holding a documented module and nothing else. Here is the outline of invoices:

```crystal source=reference/app/billing/invoices.cr
# app/billing/invoices.cr
# An invoice is an organization's request that a customer pay for what
# it delivered: lines of work or goods, priced in one currency.
#
# It starts as a draft, which the organization may change or discard
# as it likes. Sending it to the customer makes it binding: it gets its
# number, falls due after the payment term, and never changes again. A
# payment of its amount settles it; a sent invoice still unpaid after
# its due date is overdue, and the customer is reminded.
#
# Invoices are what the rest of billing turns on: a customer owes what
# its sent invoices add up to, and payments exist to settle them.
module Billing::Invoices
  # How long a customer has to pay a sent invoice.
  PAYMENT_TERM = 14.days
end
```

An outline says, in plain words, what the concept means in this app, what it is for, what it guarantees, and how it relates to its neighbours. It is the first thing a person or an agent reads before changing the namespace, and the place to change first when the meaning changes. Constants that belong to the concept as a whole, such as `PAYMENT_TERM`, may live there too.

An outline is not a place for code. A namespace without one, or an outline that holds other types or methods, is reported by `Punch/NamespaceFile`.

## Records, models, and stores

An entity's data takes up to three forms, each in its own file:

- **The record** is generated from the schema and `columns.cr` into `records.gen.cr`: a struct with one field per column, already in domain types, and a `Table` with find, insert, update, and delete. Behaviour that belongs to a row goes in `record.cr`, which reopens the generated struct:

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

- **A model** with states is a file named after its type. Invoices have a lifecycle, so `invoice.cr` defines `Invoice::Draft`, `Invoice::Sent`, and `Invoice::Paid`, each its own immutable type, with transitions as methods.
- **A store** (`store.cr`) loads rows into the model's types and saves them back.

Which form an entity needs is decided in [Domain Models](domain-models.md).

## Requires are generated

You never write a `require` for a file under `app/`. The app's library file requires Punch and one generated file:

```crystal source=reference/src/invoicing.cr
# src/invoicing.cr
# The app's code, without the command line, so specs can require it.
# `app/requires.gen.cr` requires every file under app/ in the order of its
# kind (lib/punch/docs/structure.md).
require "punch"
require "../app/requires.gen"
```

`punch requires:generate` writes `app/requires.gen.cr`, which requires every file under `app/` in the order of its kind: types before the events that carry them, columns before the generated records, records before the queries that return them, operations before the pages that call them, roles last because they name operations. Within a kind, shallower files come first. `punch dev`, `punch typecheck`, and `punch check` keep the file current, so a new file is part of the next build without your doing anything.

Every file must be of a known kind, by its name or its directory. A file that fits none is refused, with the list of kinds:

```console
$ punch requires:generate
punch: app/billing/disputes/helpers.cr:1: no file kind fits helpers.cr: it holds Disputes but is not named after one of them (disputes.cr); name it after the type it holds, or make it a file of another kind
Each file under app/ is one of these, required in this order (lib/punch/docs/structure.md):
  translations.gen.cr, from punch i18n:generate
  types.cr: IDs and domain primitives
  events.cr: the namespace's domain events
  outlines: X.cr beside the directory X/
  columns.cr: an entity's column types
  records.gen.cr, from punch db:generate
  files named after the type they hold (invoice.cr holds Invoice)
  setup.cr in app/: every Punch.* declaration and Punch::*.config, after the types it names
  record.cr: behaviour of a generated record
  queries.gen.cr, from punch db:generate
...
```

There is no `helpers.cr` or `utils.cr` kind on purpose. A helper is a type with a name, and its file is named after it.

## Specs mirror `app/`

A spec file sits at the path of the file it tests, under `spec/` and without the kind directory: `spec/billing/invoices/send_spec.cr` for `app/billing/invoices/operations/send.cr`, `show_page_spec.cr` for a page, `invoice_spec.cr` for the model. Journeys across namespaces, such as signing up and then sending a first invoice, go in `spec/journeys/`. Factories per slice go in `spec/support/<slice>.cr`. See [Testing Punch Applications](testing-punch-applications.md).

## Adding a namespace or an entity

`punch generate` writes the files that follow from the structure, never the content. For a namespace without tables:

```console
$ punch generate namespace billing/disputes
  created  app/billing/disputes.cr
  done    requires:generate

Next:
  Write the outline in app/billing/disputes.cr.
  Add the namespace's operations under app/billing/disputes/operations/ and pages under app/billing/disputes/pages/ (lib/punch/docs/structure.md).
  punch check
```

The new outline starts with a placeholder:

```crystal illustrative="the file punch generate writes; it is not part of the reference app"
# punch generate: say here what Billing::Disputes means in this app, what it is
# for, and how it relates to the other namespaces, in plain words
# (lib/punch/docs/models.md). `punch check` reports this line until then.
module Billing::Disputes
end
```

`punch check` fails until you replace the placeholder with the meaning, which is the point: a namespace nobody can explain is not ready.

For an entity, `punch generate entity billing/plans` also writes `types.cr` with `Billing::Plans::ID`, a `columns.cr` that maps the `plans` table's ID, foreign keys, and `_at` columns, and a `queries.sql` to start from. If the table does not exist yet, it writes a migration to fill in first. The full behaviour is in the [tooling reference](../tooling.md#punch-generate).

There is no generator for operations, pages, or SQL. Those are the app's content, and the reference app and your own existing slices are the pattern to copy.

## The checks that keep the layout

| Rule | Check | Message |
|---|---|---|
| One file per operation, page, job, mail, and subscriber, named after it, in its kind's directory | `Punch/OperationFile` | `` `Billing::Invoices::Send` belongs in `billing/invoices/operations/send.cr`: one file per operation, page, job, mail, and subscriber, named after it (lib/punch/docs/structure.md) `` |
| Every operation names its answers | `Punch/OperationOutput` | `` `Send` declares what `call` returns as `alias Output = A \| B`, above `call` (lib/punch/docs/operations.md) `` |
| A file's outermost module is its directory's namespace | `Punch/NamespaceFile` | `` `Billing::Disputes` is outside `Billing::Invoices`, the namespace of this file's directory: a file's outermost module is its directory's; move it to `…` (lib/punch/docs/structure.md) `` |
| Every namespace has an outline, and an outline is only a documented module | `Punch/NamespaceFile` | `` `Billing::Disputes` is a namespace: outline it in `app/billing/disputes.cr`, a documented `module Billing::Disputes` (lib/punch/docs/models.md) `` |
| A record's behaviour lives in `record.cr` | `Punch/RecordFile` | `` `Billing::Customers::Record` belongs in `…/billing/customers/record.cr`, the file named after it, where the generated record points (lib/punch/docs/models.md) `` |
| App-wide declarations live in `app/setup.cr` | `Punch/SetupFile` | `` `Punch.assets` configures the whole app: put it in `app/setup.cr`, with every Punch declaration (lib/punch/docs/structure.md) `` |
| Every file is of a known kind | `requires` step | `no file kind fits …` |

They run in `punch check`, as the `lint` and `requires` steps. Each message names the right place, so the fix is to move the code there.

## Common mistakes

**A `helpers.cr` or `utils.cr`.** It fits no file kind, and `requires:generate` refuses it. Give the helper a name that says what it is, make it a type or a module function, and name its file after it (`sharing.cr` holds `Billing::Invoices::Sharing`).

**Several operations in one file.** It compiles, but `Punch/OperationFile` reports the second one with the file it belongs in. A trace that names `Billing::Invoices::Destroy` must lead to `destroy.cr`.

**Nesting a module per directory level by hand.** Writing `module Billing; module Invoices; module Operations; struct Send` puts `Send` in a namespace named `Operations`, which changes its route and its name everywhere. Kind directories add no namespace; write `module Billing::Invoices` at the top of the file.

**An operation directly in a slice.** `Billing::Export` in `app/billing/operations/export.cr` has no entity to belong to. Put it with the entity whose state it reads or writes, or give the use case a namespace of its own.

**Reaching into another slice's tables.** A page in billing that runs `Accounts::Users::Table.find` ties billing to how accounts stores users. Go through accounts' operations, or ask its policies.

**Leaving the outline placeholder.** `punch check` keeps failing on the `# punch generate:` line. Write what the namespace means; one honest paragraph is enough.

**Settings in a type's file.** `Punch.assets "assets"` at the top of a layout's file works, until someone looks for it in `app/setup.cr`. `Punch/SetupFile` reports it.

## Limits

- Punch cannot check that a slice uses another only through its operations. It is listed under "Rules no tool checks yet" in the app's `AGENTS.md`.
- `punch generate` does not write operations, pages, or SQL, and does not move existing files.

## Rules

- Name a thing first; its name decides its file, route, and permission. ([The shape of an app](#the-shape-of-an-app))
- One file per operation, page, job, mail, and subscriber, named after it, in its kind's directory. ([Where each thing goes](#where-each-thing-goes))
- Put a use case with the entity whose state it changes; a use case that writes several gets a namespace of its own. ([A use case goes where it writes](#a-use-case-goes-where-it-writes))
- Nest a namespace only under its owner. ([Slices, namespaces, and entities](#slices-namespaces-and-entities))
- Every namespace has an outline that says in plain words what it means; update it when the meaning changes. ([Outlines: what a namespace means](#outlines-what-a-namespace-means))
- Never write a `require` for a file under `app/`, and never edit `app/requires.gen.cr`. ([Requires are generated](#requires-are-generated))
- App-wide declarations go in `app/setup.cr`. ([Where each thing goes](#where-each-thing-goes))
- Another slice's data is reached only through its operations. ([Slices, namespaces, and entities](#slices-namespaces-and-entities))

## Reference

- [Application structure](../structure.md): slices, file kinds and their order, derived names.
- [Tooling](../tooling.md#punch-generate): `punch generate` in full.
- [Domain models](../models.md): outlines, records, state types, and stores.
- [Lint rules](../reference/lint-rules.md): every Punch rule and its message.
