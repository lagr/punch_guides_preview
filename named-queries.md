# Named Queries and Generated Code

Applies to Punch 0.1.

Punch has no ORM and no query builder. An app's SQL lives in `.sql` files, one per entity, and `punch db:generate` turns each named query into a typed Crystal method: its parameters become keyword arguments in domain types, its result a row type or the table's record. Every table an entity owns also gets a generated record and a `Table` module to find, insert, update, and delete rows. The generated files are committed, so every database call can be read, searched, and reviewed in a diff.

After reading this guide, you will know:

- how to write a named query, and call the method generated from it;
- how column types become domain types, once per column;
- what the generated record and `Table` give you for each table;
- how a unique index becomes a typed result instead of an exception;
- what the generator refuses, and how its errors read;
- how to read a generated file when you need to.

Before you start: you know [SQLite in Punch](sqlite-in-punch.md) and [Migrations](migrations.md). The examples come from the reference app's customers, in `reference/app/billing/customers/`.

## A named query

Each entity keeps its SQL in `queries.sql` in its directory. Here is the reference app's customer search:

```sql source=reference/app/billing/customers/queries.sql select=Search
-- app/billing/customers/queries.sql
-- name: Search :many
-- One organization's customers whose name starts with the prefix, in any
-- case and script, sorted by name in the column's collation.
SELECT * FROM customers
WHERE organization_id = :organization_id AND name LIKE :prefix || '%' ESCAPE '\'
ORDER BY name, id
LIMIT :limit;
```

- **The header** is `-- name: Name :kind`. The method is the name in snake_case: `search`.
- **The kind** says what the method returns: `:one` the first row or `nil`, `:many` an array, `:exec` nothing, `:execrows` the number of rows changed. `:page` is a list that pages by key ([the queries reference](../queries.md#pages-keyset-pagination)).
- **Comment lines** right after the header become the method's documentation.
- **Parameters** are named (`:organization_id`, `:prefix`) and become keyword arguments. A parameter used twice is passed once. `?` is refused.
- **One statement per query.** Transaction statements are refused, because a `Writer` is one transaction already.

`punch db:generate` writes `queries.gen.cr` beside the file, with this method:

```crystal illustrative="the signature punch db:generate writes into reference/app/billing/customers/queries.gen.cr"
def self.search(db : Punch::Reader | Punch::Writer, *, organization_id : Accounts::Organizations::ID, prefix : Punch::LikeText, limit : Int64) : Array(Billing::Customers::Record)
```

The Query that searches customers calls it like any method:

```crystal source=reference/app/billing/customers/operations/search.cr select=Billing::Customers::Search#call
# app/billing/customers/operations/search.cr
module Billing::Customers
  # ...

  # Finds the customers of the organization the actor works in whose name starts with `q`, in any case and script
  # (`ä` finds `Ärger`), sorted by name as German readers expect.
  @[Punch::Authorize(Accounts::Organizations::Current)]
  struct Search < Punch::Query
    # ...

    def call(input : Input, db : Punch::Reader, actor : Invoicing::Actor) : Output
      # The prefix is text, not a pattern: `%` and `_` in it match themselves.
      prefix = Punch::LikeText.new(input.q.value)
      Matches.new(Queries.search(db, organization_id: actor.organization_id, prefix: prefix, limit: input.limit.to_i.to_i64).map { |record| Customer.new(record) })
    end
  end
end
```

Everything about the call is checked at compile time: the parameter names, their types (an `Accounts::Organizations::ID`, not any integer), and the result. `punch dev` and `punch typecheck` regenerate the file before each build, so a changed query is a changed method at once.

## How the generator knows the types

Every query is prepared against `db/schema.sql` in the same SQLite the app links, and the generator works out each type from the schema:

- **Result columns** that come straight from a table take its type and nullability. Columns on the optional side of a `LEFT JOIN` are nilable.
- **Expressions** are typed by rules: `count` is an integer; `sum`, `min`, `max`, and `avg` are nilable, since they are `NULL` over no rows; `coalesce`, `CASE`, `CAST`, arithmetic, and string functions have their own rules.
- **Parameters** take the type of what they are compared with, inserted into, or assigned to. `:organization_id` above is compared with `customers.organization_id`, so it is an `Accounts::Organizations::ID`. A parameter of `LIKE` is a `Punch::LikeText`, which escapes `%` and `_` ([Keeping SQL Safe and Fast](keeping-sql-safe-and-fast.md)).
- **Annotations** cover the rest, in the header: `-- column cents: Int64`, `-- param ids: String`.

The result of `Search` is every column of `customers`, in order, so it returns the table's `Record` instead of a row type of its own. A narrower `SELECT` gets `Search::Row` with only its columns.

A mistake in the SQL is found when you generate, not when the query first runs, with SQLite's message and the file, line, and column:

```console
$ punch db:generate
punch: app/billing/invoices/queries.sql:6:27: query Overdue: no such column: due_on
```

All mistakes of a run are reported together, and nothing is written until they are fixed.

## Column types, once: `columns.cr`

SQLite stores integers, text, and so on. The app means an `Accounts::Organizations::ID`, a `Punch::Email`, a `Time`. Each entity says which column holds which domain type, once, in its `columns.cr`:

```crystal source=reference/app/billing/customers/columns.cr
# app/billing/customers/columns.cr
# Which columns of these tables hold which domain types (lib/punch/docs/queries.md).
# Read by `punch db:generate` and required by the app.

Punch.table "customers", Billing::Customers,
  id: Billing::Customers::ID,
  organization_id: Accounts::Organizations::ID,
  name: Punch::NonEmptyString,
  company_name: Punch::NonEmptyString,
  email: Punch::Email,
  created_at: Time,
  locale: Punch::LocaleTag
```

Every generated query and record uses these types: a parameter compared with `customers.email` is a `Punch::Email`, and a row read back has `email : Punch::Email`. The second argument is the module that gets the table's `Record` and `Table`.

- A column maps to a domain primitive (typed IDs included), `Time`, or `Bool`. Columns not listed keep their stored type.
- Only `STRICT` tables can be mapped, so the stored types are enforced by SQLite.
- `Time` is stored as text in UTC with nanoseconds, at a fixed width, so it sorts in time order.
- **Rows are parsed when they are read.** A stored value that does not parse into its type raises `Punch::Database::RowError` with the table, column, and row, never the value. That makes tightening a type a two-step change: audit the stored values first with `db:audit` ([Keeping SQL Safe and Fast](keeping-sql-safe-and-fast.md)).

`punch generate entity` writes a first `columns.cr` for a new entity, mapping its ID, foreign keys, and `_at` columns, and listing the others for you to map ([Application Structure](application-structure.md#adding-a-namespace-or-an-entity)).

## Records and `Table`

For each mapped table, `punch db:generate` writes `records.gen.cr` beside `columns.cr`: a `Record` with one field per column, and a `Table` module.

```crystal illustrative="the shape of reference/app/billing/customers/records.gen.cr, bodies left out"
module Billing::Customers
  record Record,
    id : Billing::Customers::ID,
    organization_id : Accounts::Organizations::ID,
    name : Punch::NonEmptyString,
    company_name : Punch::NonEmptyString?,
    email : Punch::Email,
    created_at : Time,
    locale : Punch::LocaleTag?

  module Table
    def self.find(db : Punch::Reader | Punch::Writer, id : Billing::Customers::ID) : Record?
    def self.insert(db : Punch::Writer, *, organization_id : Accounts::Organizations::ID, name : Punch::NonEmptyString, company_name : Punch::NonEmptyString? = nil, email : Punch::Email, created_at : Time, locale : Punch::LocaleTag? = nil) : Record | EmailTaken
    def self.update(db : Punch::Writer, record : Record) : Record | EmailTaken?
    def self.delete(db : Punch::Writer, id : Billing::Customers::ID) : Bool
  end
end
```

- **Records are immutable.** Change one with `copy_with` and write it with `update`, which sets every column. `Customers::Update` does exactly that.
- **`insert` leaves out the rowid**, which SQLite assigns, and takes a column's default when given `nil`.
- **Behaviour goes in `record.cr`**, which reopens the record ([Domain Models](domain-models.md#tier-1-behaviour-on-the-generated-record)).
- **A query cannot write without a `Writer`.** Methods that write take `Punch::Writer`, so a Query, which has a `Reader`, cannot call them.

## Uniqueness as a typed result

A unique index is the one rule only the database can decide. So a write that can break one returns it as a value instead of raising. `customers` has a unique index on the organization and the email, and the generator wrote a type for it:

```crystal illustrative="from reference/app/billing/customers/records.gen.cr"
# Another `customers` row has this organization_id, email. Returned by the writes that can violate it.
struct EmailTaken
  include Punch::Database::Violation

  CONSTRAINT = "customers.organization_id, customers.email"
  FIELD      = "email"
  MESSAGE    = "UNIQUE constraint failed: customers.organization_id, customers.email"
end
```

`Table.insert` returns `Record | EmailTaken`, and the compiler makes the Command handle both:

```crystal source=reference/app/billing/customers/operations/create.cr select=Billing::Customers::Create#call
# app/billing/customers/operations/create.cr
module Billing::Customers
  # ...

  # Adds a customer to the organization the actor works in. Each
  # organization's customers have unique email addresses.
  @[Punch::Authorize(Accounts::Organizations::Current)]
  struct Create < Punch::Command
    # ...

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

Returning the violation answers 422 with `taken` at `email`, in the API and as a field error on the form. No lookup before the insert: the Command holds the write lock, and the index decides. Where a write cannot violate by construction, `Punch.cannot_violate(write)` drops the violation from the type and raises if one happens anyway. See [Results, Errors, and Status Codes](results-errors-and-status-codes.md#field-errors-rejections-and-violations).

Other constraint failures, such as foreign keys and `CHECK`, still raise, because they mean the Command should have checked first. Rescuing them in app code is reported by `Punch/ConstraintRescue`.

## Reading a generated file

Generated files are meant to be read. Each method calls one statement, with its SQL as a constant beside it and its trace span named after the method:

```crystal illustrative="from reference/app/billing/customers/queries.gen.cr"
# Which organization a customer belongs to, for the policy.
def self.organization(db : Punch::Reader | Punch::Writer, *, customer_id : Billing::Customers::ID) : Organization::Row?
  db.query_one?(Organization::SQL, Punch::Stored.dump(customer_id), name: "Billing::Customers::Queries.organization") { |row| Organization::Row.read(row) }
end
```

So a slow `Billing::Customers::Queries.organization` in a trace or the ops console leads straight to its `-- name: Organization` in `queries.sql`. `punch db:generate --json` lists every query with its kind, result type, and each column's and parameter's type with where the type came from.

Never edit a generated file. Change the SQL, the schema, or `columns.cr`, and run `punch db:generate`; `punch check` fails on a stale one.

## Testing queries

Query methods are plain methods over a database, so a spec writes rows with the generated inserts and calls the query:

```crystal source=reference/spec/billing/customers/queries_spec.cr select="customers API reads names in German order from the collated index"
# spec/billing/customers/queries_spec.cr
describe "customers API" do
  # ...

  # Phase 1 exit: a list of names with umlauts sorts correctly through an
  # index, not by sorting the result.
  it "reads names in German order from the collated index" do
    Punch::Spec.db.write do |db|
      organization = AccountsSpec.organization(db, AccountsSpec.user(db).id)
      %w[Zander Ärger Abel Öhler Ober Müller Mueller].each_with_index do |name, index|
        Punch.cannot_violate(Billing::Customers::Table.insert(db, organization_id: organization, name: InvoicingSpec.parse(Punch::NonEmptyString, name),
          email: InvoicingSpec.parse(Punch::Email, "x#{index}@example.org"), created_at: Time.utc))
      end
      db.query_all("SELECT name FROM customers ORDER BY name", &.read(String)).should eq(%w[Abel Ärger Mueller Müller Ober Öhler Zander])
      plan = db.query_all("EXPLAIN QUERY PLAN #{Billing::Customers::Queries::Search::SQL}", organization.value, "", 20_i64) do |row|
        3.times { row.read(Int64) }
        row.read(String)
      end
      plan.join(" | ").should contain("customers_by_name")
      plan.join(" | ").should_not contain("TEMP B-TREE")
    end
  end
end
```

Most queries are covered by the endpoint specs of the operations that use them. Write a query spec where the SQL itself is the subject: an order, a collation, a page boundary.

## Common mistakes

**Building SQL from strings.** `"SELECT * FROM customers ORDER BY #{column}"` is refused by `Punch/SQLText`, and is how injection starts. Choices of filter and order are declared in the query ([Keeping SQL Safe and Fast](keeping-sql-safe-and-fast.md)).

**Looking up before inserting.** `Table.find_by_email` then `insert` is a race. Insert, and handle the violation.

**Mapping the same column in two places.** Each table has one owner, and its mapping lives in that entity's `columns.cr`. A table of another namespace is refused with the file it belongs in.

**`SELECT *` for a narrow list.** It returns the record, every column. A list that shows three columns reads three: select them, and the query gets its own `Row`.

**Editing `queries.gen.cr` to fix a type.** The next run overwrites it. Map the column in `columns.cr`, or annotate the query's header.

**Rescuing `ConstraintError`.** `Punch/ConstraintRescue` reports it. A unique violation comes back as a value; any other constraint failure is a check the Command should make first.

## Limits

- No query builder and no dynamic SQL. Optional filters and choices of order are declared in the query ([the queries reference](../queries.md#filters-and-sorting-declared-clauses)).
- `RIGHT` and `FULL JOIN` need every column annotated, since their nullability is not derived.
- Foreign key failures cannot be told apart: SQLite does not say which key failed.

## Rules

- Write SQL in the entity's `queries.sql`, one named statement per query. ([A named query](#a-named-query))
- Map each column to its domain type once, in the owning entity's `columns.cr`. ([Column types, once: `columns.cr`](#column-types-once-columnscr))
- Let unique indexes decide uniqueness, and return their violations; never rescue `ConstraintError`. ([Uniqueness as a typed result](#uniqueness-as-a-typed-result))
- Never edit a generated file; change its source and run `punch db:generate`. ([Reading a generated file](#reading-a-generated-file))
- Never build SQL text from values. ([Common mistakes](#common-mistakes))

## Reference

- [Queries](../queries.md): named queries, types, `columns.cr`, records, violations, and every generator option.
- [Persistence](../persistence.md#reading-and-writing): readers, writers, and raw statements.
- [Domain models](../models.md): behaviour on records.
