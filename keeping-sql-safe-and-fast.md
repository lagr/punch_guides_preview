# Keeping SQL Safe and Fast

Applies to Punch 0.1.

Punch has no ORM: an app's database access is SQL it writes itself. That gives full control over every query, and two ways to get it wrong that an ORM would hide: a value pasted into SQL text, which opens the door to SQL injection, and a query that reads every row of a table, which is fast in development and slow in production. `punch check` refuses both, before the code ships. This guide explains each refusal and the honest fix.

After reading this guide, you will know:

- why a value never goes into SQL text, and how Punch enforces it;
- how to let a request choose a sort order or filter without building SQL;
- why `punch check` refuses a query plan, and how to fix the query rather than silence the check;
- how to tighten a domain type over data that is already stored.

Before you start: you know how named queries work ([Queries](../queries.md#named-queries)), and how to write a migration ([Migrations](migrations.md)).

## No value in SQL text

Every value reaches SQLite as a bound parameter. The text of a statement comes from the source code alone, never from a request, a row, or a variable. Then no input, however crafted, can change what a statement does.

In named queries, this is how the language works: `:customer_email` in a `.sql` file is a parameter, and the generated method binds it. Hand-written SQL in Crystal, which apps rarely need, is checked by the lint rule `Punch/SQLText`, part of `punch check`. It accepts a string literal, a constant, and text built from those only. It reports anything else, with this message:

```crystal illustrative="does not pass punch check; never build SQL like this"
db.query_all("SELECT id FROM invoices WHERE number = '#{number}'") { |row| row.read(Int64) }
```

```text
SQL text from a value: pass a literal or a constant, or build the statement from constants only, and bind every value as a parameter (lib/punch/docs/queries.md#sql-text-is-never-built-from-values)
```

The fix binds the value:

```crystal illustrative="hand-written SQL with a bound value; a named query is better still"
db.query_all("SELECT id FROM invoices WHERE number = ?", number) { |row| row.read(Int64) }
```

This applies to IDs, numbers, and limits too. A value that "cannot be a string" today is the one a later change makes a string.

## Choices without SQL text

A list that the user sorts or filters is where SQL text is most tempting to build: `ORDER BY #{params[:sort]}`. In Punch, the choices are declared in the query instead, and the request picks among them:

```sql source=reference/app/billing/customers/queries.sql select=Page
-- app/billing/customers/queries.sql
-- name: Page :page
-- One organization's customers, a page at a time: in the order they were
-- created, or by name in the column's collation (through
-- customers_by_name).
SELECT * FROM customers
WHERE organization_id = :organization_id
-- order created: ORDER BY id
-- order name: ORDER BY name, id
```

`-- order created:` and `-- order name:` declare the two sorts. The generator turns them into an enum, `Queries::Page::Order`, which an operation's input takes as a field and the request names as `?order=name`. A name that is not declared fails to parse, like any other input. Optional filters are declared the same way, with `-- when` clauses ([Queries: declared clauses](../queries.md#filters-and-sorting-declared-clauses)).

## Searching for user text

A user's search text is a value too, and in a `LIKE` it has a second danger: `%` and `_` are wildcards. A search for "50%" should find "50% Rabatt", not every name that starts with "50". So:

- build the pattern in SQL around the parameter, `name LIKE :prefix || '%' ESCAPE '\'`;
- pass the text as a `Punch::LikeText`, which escapes the wildcards when it is bound.

The generator refuses a `LIKE` with a parameter and no `ESCAPE '\'`, and types that parameter as `Punch::LikeText`, so passing a plain `String` does not compile. The reference app's customer search shows both halves ([Text and Sort Order](text-and-sort-order.md#searching-lower-upper-and-like)). Full-text search with FTS5 works the same way, with `Punch::FtsPhrase` ([Queries: filtering by text](../queries.md#filtering-by-text)).

## Query plans: no full scans, no big sorts

A query that reads every row of a table is fine with 50 rows and a problem with five million. Punch asks SQLite how it would run each named query, `EXPLAIN QUERY PLAN` from the schema, and fails on two kinds of step:

- a **scan**: reading every row of a table;
- a **sort**: sorting every row it read before returning the first.

Suppose the examples app gets a query for one customer's invoices:

```sql illustrative="a new query that punch check refuses"
-- name: ByCustomer :many
-- The invoices of one customer, newest first.
SELECT id, number FROM invoices
WHERE customer_email = :customer_email
ORDER BY due_at DESC;
```

No index serves it, so `punch check` fails on its `explain` step:

```console
$ punch db:explain --check
punch: app/billing/invoices/queries.sql:13:1: query ByCustomer reads every row of invoices. Add an index that serves it, or say why it is fine in the header: `-- scan: <reason>`.
app/billing/invoices/queries.sql:13:1: query ByCustomer sorts every row it reads for ORDER BY. Add an index that serves it, or say why it is fine in the header: `-- sort: <reason>`.
```

The fix is an index on what the query filters by, then what it sorts by, in a new migration:

```sql illustrative="the migration that fixes it"
-- One customer's invoices by due date.
CREATE INDEX invoices_by_customer ON invoices (customer_email, due_at);
```

```console
$ punch db:explain --check
3 queries, 3 statements: no full scans or sorts
```

An index serves both directions, so `DESC` needs nothing more. A query with declared clauses is explained in every shape a request can give it, each filter and each order, and a finding names the shape.

### When a scan or sort is fine

Some are: a sort of the few rows one user has, a scan of a table that holds one row per currency. Then the query's header says why, and `punch check` accepts that kind of step for that query:

```sql illustrative="the reference app's statement query, which groups one customer's invoices"
-- name: Statement :many
-- One customer's sent invoices per currency.
-- sort: one customer's sent invoices, grouped by currency
SELECT ...
```

`punch db:explain` lists every accepted step with its reason:

```console
$ punch db:explain
20 queries, 43 statements: 2 with a full scan or sort (2 accepted)
app/accounts/organizations/queries.sql:10:1 Mine
  sort  sorts every row it reads for ORDER BY; accepted: the organizations of one user, a few rows
app/billing/customers/queries.sql:22:1 Statement
  sort  sorts every row it reads for GROUP BY; accepted: one customer's sent invoices, grouped by currency
```

The reason must be true of the data, not of today's test database. A `-- scan:` added only to make the check pass is a slow query shipped on purpose.

The plans come from the schema without data, as a new database has them. In production, SQLite gathers statistics and may choose other plans; the check catches queries no index can serve, not every slow one. The ops console's performance view shows the named queries that take longest in production ([Ops data](../ops.md)).

## Tightening a type over stored data

Rows are parsed into their domain types when they are read. So making a type stricter, such as a shorter maximum for a name or a stricter rule for an invoice number, can turn rows that were valid yesterday into rows that fail to read, and the operations that read them into errors. Find them before shipping the change.

`db:audit` parses every mapped column of every row against the types the binary has. It is a command of the app's binary, which has the types, and it takes the path of any copy of the database:

```console
$ bin/examples db:audit /tmp/restored/primary.db
/tmp/restored/primary.db: 1 rows in 1 tables, every value parses.
```

When values fail, it names the column, the type, the reason code, the count, and the first row IDs, never the values themselves:

```text illustrative="the shape of a finding"
customers.email: 2 values do not parse as Punch::Email (2 invalid_email); rows 17, 210
```

So a type change is two steps:

1. Change the type, build the binary, and run `db:audit` against a restored copy of production ([Backup and Restore](backup-and-restore.md)).
2. Fix the rows it names, with a migration or through a Command, and run it again until it is clean. Then ship the type.

Never loosen the audit or coerce values when reading to make it pass: the rows are wrong, and the next reader would trip over them. Punch also audits the primary database every night at 03:00 UTC and reports columns that do not parse in the ops console.

## Common mistakes

**Interpolating "just an ID".** `"… WHERE id = #{id}"` is reported like any other value. Bind it.

**A sort column from the request.** `ORDER BY #{params[:sort]}` does not pass the lint rule, and an allow-list in Crystal is a second place to keep in sync. Declare `-- order` clauses.

**A `LIKE` on raw user text.** The generator refuses `LIKE :q` without `ESCAPE '\'`, and a plain `String` for a `Punch::LikeText` parameter does not compile.

**Silencing `explain` with `-- scan:`.** It moves a slow query into production. Add the index, or change the query; write `-- scan:` only with a reason that holds for production data.

**An index for every finding.** Each index slows writes and takes space. An index on a filter followed by the sort usually serves a whole family of queries; check `punch db:explain` after adding one.

**Tightening a type without an audit.** The first request that reads an old row fails. Audit a copy first.

## Limits

- Plans are checked without data or statistics; a query that is slow because of the data's shape passes.
- `Punch/SQLText` checks the methods that run SQL. SQL text built far from them and passed through other methods is checked only where it is run.
- `db:audit` checks mapped columns of tables with a rowid.

## Rules

- Never put a runtime value into SQL text; bind every value, IDs and limits included. ([No value in SQL text](#no-value-in-sql-text))
- Let a request choose sorts and filters only among declared `-- order` and `-- when` clauses. ([Choices without SQL text](#choices-without-sql-text))
- Pass user text to `LIKE` as a `Punch::LikeText`, with `ESCAPE '\'`. ([Searching for user text](#searching-for-user-text))
- Fix a full scan or sort with an index or a better query; accept one with `-- scan:` or `-- sort:` only for a reason true of production data. ([Query plans: no full scans, no big sorts](#query-plans-no-full-scans-no-big-sorts))
- Before tightening a domain type, run `db:audit` on a copy of production and fix the rows it names; never loosen the audit. ([Tightening a type over stored data](#tightening-a-type-over-stored-data))

## Reference

- [Queries](../queries.md): named queries, declared clauses, pages, the SQL text rule, query plans, and the audit.
- [Text and Sort Order](text-and-sort-order.md): collations and `LIKE` beyond ASCII.
- [Migrations](migrations.md): adding the index.
