# Migrations

Applies to Punch 0.1.

A Punch app's schema changes through migrations: plain SQL files in `db/migrations/`, applied in order, each exactly once. There is no schema DSL to learn and no down migration to keep in sync. Punch records what ran, writes the resulting schema to `db/schema.sql` for review, tells you what each statement costs on a large table, and applies pending migrations when a new release boots.

After reading this guide, you will know:

- how to add a table, a column, an index, and a constraint, and what each costs on a table with millions of rows;
- how to change a column safely across two releases, so the release before keeps working;
- why you never edit an applied migration or `db/schema.sql`, and what `punch check` says if you do;
- how migrations reach production.

Before you start: you know where the app's data lives ([SQLite in Punch](sqlite-in-punch.md)).

## Writing a migration

`punch db:migration NAME` creates an empty file with a timestamp:

```console
$ punch db:migration create_invoices
Created db/migrations/20261023133847_create_invoices.sql
```

The file holds plain SQL, any number of statements. The examples app's invoices table:

```sql source=docs/guides/examples/db/migrations/20261023133847_create_invoices.sql
-- db/migrations/20261023133847_create_invoices.sql
-- Invoices, as much of them as the guides' job examples need: who pays,
-- when the invoice is due, whether it is paid, and when its customer was
-- last reminded.
CREATE TABLE invoices (
  id             INTEGER PRIMARY KEY,
  number         TEXT    NOT NULL UNIQUE,
  customer_email TEXT    NOT NULL,
  due_at         TEXT    NOT NULL,
  paid_at        TEXT,
  reminded_at    TEXT
) STRICT;

-- The unpaid ones by due date, for the overdue sweep.
CREATE INDEX invoices_unpaid ON invoices (due_at) WHERE paid_at IS NULL;
```

- Tables are `STRICT`, so SQLite enforces the declared types. A table that a named query maps must be.
- Comments say why the table or index exists. The migration is the one place that records the reason a column is there.
- Punch runs the whole file in one transaction, so it must not contain `BEGIN`, `COMMIT`, or `ROLLBACK`. A statement that fails rolls the whole migration back, and the error names the file, line, and column.

Then apply it to the development database:

```console
$ punch db:migrate
Applied 20261023133847_create_invoices.sql
Wrote db/schema.sql
```

`punch dev` does the same on every change, and `punch generate entity` maps a new table for named queries ([Tooling](../tooling.md)).

## `db/schema.sql`

Every `punch db:migrate` rewrites `db/schema.sql` from the migrated database: every table, index, view, and trigger as SQLite stores it, in a stable order, followed by the applied versions. It is the one file that shows the whole database, so a code review sees exactly what a migration changed. Specs start from it, and the query generator reads it.

Commit it with the migration, and never edit it. `punch check` builds a fresh database from the migrations and compares the result with the committed file (its `schema` step), so a hand edit or a forgotten migration fails the build.

## Forward only

Migrations only go forward. There is no down migration:

- **In development,** delete the database and migrate again, or fix the migration before you commit it.
- **In production,** a mistake is fixed by a new migration, or by restoring to a point before it ([Backup and Restore](backup-and-restore.md#restoring-to-a-point-in-time)).

Once a migration has been applied anywhere, it is never edited. Punch records each migration's SHA-256, and refuses to migrate while an applied one has changed:

```text
applied migrations were changed afterwards: 20261009090000_sort_customer_names. Restore them and put the change in a new migration.
```

A down migration would be a second, untested program that has to undo the first exactly, including for data written in between. Punch keeps what it can guarantee: a migration that fails rolls back completely, and a restore goes back to any point.

## What a migration costs

A migration runs while it holds the database's single write lock. On an empty development database everything is instant; on a table with millions of rows, some statements read or copy every row while every write in the app waits. `punch db:cost` sorts each statement into one of three classes:

| Class | Means | Statements |
|---|---|---|
| `instant` | changes the schema only | `CREATE TABLE`, `CREATE VIEW`, `CREATE TRIGGER`, renames, `DROP NOT NULL`, `ADD COLUMN` on a table that is not `STRICT` |
| `scan` | reads every row once and keeps the table | `CREATE INDEX`, `SET NOT NULL`, `ADD CHECK`, `ADD COLUMN` on a `STRICT` table, `UPDATE`, `DELETE` |
| `rebuild` | writes a new copy of every row | creating a new table, copying, dropping the old one, and renaming; `DROP COLUMN` |

A scan of a million small rows takes tens of milliseconds. A rebuild takes longer, needs room for a second copy, and holds the lock throughout. The reference app's history has three rebuilds:

```console
$ punch db:cost
15 migrations: 9 instant, 3 scan, 3 rebuild
instant  20261008200000_create_customers.sql
instant  20261008200100_create_invoices.sql
rebuild  20261009090000_sort_customer_names.sql  rebuilds customers (not acknowledged)
  rebuild   13  INSERT INTO customers_new (id, name, company_name, email, created_at)
                copies every row of customers into its new table
  rebuild   16  DROP TABLE customers
                drops customers, which a new table replaces
  rebuild   17  ALTER TABLE customers_new RENAME TO customers
                renames the new table to customers
  scan      19  CREATE INDEX customers_by_name ON customers (name)
                builds the index from every row
...
```

**A new rebuild must be acknowledged** in its file, with a line naming each table it rebuilds:

```sql illustrative="the start of a rebuild; SQLite cannot change a column's collation in place"
-- Customer names sort as German readers expect.
-- rebuild: customers
CREATE TABLE customers_new (
  ...
```

`punch check` fails on a rebuild without that line, and on a line naming a table the migration does not rebuild. It checks only the migrations that the main branch does not have yet, which is why the reference app's early rebuilds, from before the rule, are listed as not acknowledged without failing.

Before writing a rebuild, look for a change that keeps the table: a unique index instead of a `UNIQUE` constraint, a trigger, `ALTER COLUMN … SET NOT NULL`, `ALTER TABLE … ADD CHECK`. SQLite 3.53, which Punch ships, can add a `CHECK` and set `NOT NULL` in place.

## The previous release keeps working

A deploy starts the new release beside the old one. The new one applies its migrations at boot, and for a few seconds the old process keeps serving requests against the new schema. After a rollback, an older binary runs against migrations it does not know, which `db:status` lists as `unknown` and which do not stop it. So every migration must leave the previous release working.

That rules out removing or renaming what the running release still uses. Change in two releases instead, expand then contract. To replace a column, for example:

```sql illustrative="release 1: add the new column beside the old one"
ALTER TABLE customers ADD COLUMN display_name TEXT;
UPDATE customers SET display_name = coalesce(company_name, name);
```

Release 1 writes both columns and reads the new one. Once it runs everywhere, release 2 removes the old column, which no running code reads any more:

```sql illustrative="release 2: drop the old column; a rebuild, so acknowledged"
-- rebuild: customers
ALTER TABLE customers DROP COLUMN company_name;
```

The same applies to tables, indexes a query relies on, and enum values stored in a column. Add first, remove a release later.

A migration that changes data, such as renaming a stored value, is a migration like any other: an `UPDATE` in a file, a scan of the rows it reads.

## Migrations in production

The app's binary carries its migrations, so a deploy needs no source tree. `app/setup.cr`, where app-wide settings go, declares them once, and the compiler embeds the files:

```crystal illustrative="the line from app/setup.cr that `punch new` writes"
Punch.migrations "../db/migrations"
```

- `serve` and `jobs` apply pending migrations at boot, before `/up` can answer 200. A migration that fails stops the boot, so the proxy never sends traffic to the new release and the old one keeps serving.
- Two processes that boot at once apply each migration once.
- `PUNCH_MIGRATE=off` makes the boot refuse while migrations are pending, for deploys that migrate in a separate step with `bin/<app> db:migrate`.
- `punch check` fails (its `boot` step) when the binary is built without its migrations.

`db:status` lists every migration as `applied`, `pending`, `changed`, or `unknown`, for the database at a path or in `PUNCH_STORAGE`.

## Punch's own tables

Some of Punch's features keep tables in `primary.db`: sessions and passwords, roles, the outbox for jobs and events, idempotency keys, and webhooks. They live in the app's migrations like its own tables, so the schema file shows them and they migrate with the app. Never write them by hand; `punch db:migration --punch` writes each from Punch's own SQL:

```console
$ punch db:migration --punch outbox
```

The parts are `auth`, `roles`, `outbox`, `idempotency`, and `webhooks`. `punch new` includes the ones its accounts slice needs. When a Punch upgrade changes one of these tables, `punch check` fails on its `punchtables` step and names the migration to write.

## Collations and `REINDEX`

An index on a column with a collation, such as `name TEXT COLLATE de`, is sorted by that collation's rules. Punch records the version of each collation the schema uses. When a Punch release changes a collation's rules, the database refuses to open until a migration with `REINDEX de;` is pending, so an index is never searched with rules other than those it was built with ([Text and Sort Order](text-and-sort-order.md)).

## Common mistakes

**Editing an applied migration.** Migrating then refuses with the message above. Undo the edit and write a new migration.

**Editing `db/schema.sql`.** `punch check` fails on the `schema` step. Change the schema with a migration and run `punch db:migrate`.

**Dropping or renaming a column the running release reads.** The old process fails between the migration and its own shutdown, and a rollback fails entirely. Expand, then contract.

**A rebuild "to tidy up" a large table.** It holds every write for as long as the copy takes. Prefer a change that keeps the table, and acknowledge a real rebuild with `-- rebuild:`.

**Adding `-- rebuild:` to make the check pass.** The line states that you know the migration copies the table and accept the write outage. Write it after checking the table's size, not before.

**Transaction statements in a migration.** `BEGIN` or `COMMIT` in the file is refused; Punch already runs it in one transaction.

**Pasting Punch's tables by hand.** Use `punch db:migration --punch PART`, so `punch check` can tell when an upgrade changes them.

## Limits

- No down migrations.
- A migration holds the write lock while it runs; there are no online schema changes beyond what SQLite does in place.
- `db:cost` judges each statement by what it does, not by the table's real size. Check large tables yourself.

## Rules

- Change the schema only with a new file from `punch db:migration NAME`, in plain SQL, without transaction statements. ([Writing a migration](#writing-a-migration))
- Never edit an applied migration or `db/schema.sql`; commit the schema file with its migration. ([Forward only](#forward-only), [`db/schema.sql`](#dbschemasql))
- Know a migration's cost with `punch db:cost`; prefer a change that keeps the table, and acknowledge each rebuild with `-- rebuild: <table>`. ([What a migration costs](#what-a-migration-costs))
- Keep the previous release working: add in one release, remove in a later one. ([The previous release keeps working](#the-previous-release-keeps-working))
- Write Punch's tables only with `punch db:migration --punch PART`. ([Punch's own tables](#punchs-own-tables))

## Reference

- [Migrations and schema.sql](../migrations.md): how a migration runs, every statement's class, the commands, and the API.
- [Deployment](../deployment.md#migrations): migrations at boot and during a handover.
- [Text](../text.md): collations and their versions.
