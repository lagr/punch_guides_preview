# Text and Sort Order

Applies to Punch 0.1.

Out of the box, SQLite orders text by its bytes: capitals before lowercase, and every name with an accented first letter after "zulu". Its `lower`, `upper`, and `LIKE` know only ASCII letters. Punch fixes this in the database itself, so a list of customers sorts the way a German or French reader expects, a search for "ä" finds "Ärger", and the index still does the work.

After reading this guide, you will know:

- how to make names with accents sort and search correctly, through an index;
- why text is normalized when it is parsed;
- what happens when a Punch release changes a collation's rules;
- which tools cannot read collated columns.

Before you start: you know how tables are created ([Migrations](migrations.md)).

## One text, one form

"é" can be written as one character or as "e" followed by a combining accent. They look the same, and without care they are different strings, so a unique index would accept both. Punch's text primitives, `Punch::NonEmptyString`, `Punch::Email`, `Punch::Slug`, and your own, normalize to Unicode NFC when they parse ([Input and Domain Types](input-and-domain-types.md#domain-primitives)). Text that enters through an input is in one form before it reaches the database.

Text that bypasses the primitives, such as a literal in a migration, is stored as written.

## Sorting: a collation in the schema

The order of a column is declared in the schema, with `COLLATE`. The reference app's customers sort by name as German readers expect:

```sql illustrative="the customers table as db/schema.sql shows it"
CREATE TABLE "customers" (
  id              INTEGER PRIMARY KEY,
  organization_id INTEGER NOT NULL REFERENCES organizations (id),
  name            TEXT    NOT NULL COLLATE de,
  ...
) STRICT;

CREATE INDEX customers_by_name ON customers (organization_id, name);
```

`ORDER BY name`, comparisons such as `name > :after`, and the index all follow the collation, so a sorted page of customers is read from the index without sorting. Never sort in Crystal what the database can sort: an `Array#sort` after the query reads every row and compares by bytes.

Punch has collations for German, English, French, Italian, Dutch, Portuguese, and Spanish (`de`, `en`, `fr`, `it`, `nl`, `pt`, `es`). They compare in four steps, each deciding only when the ones before are equal:

1. letters, with case and accents folded away: `Ärger` sorts with `arger`, `ß` as `ss`;
2. accents: unaccented first, `apfel` before `äpfel`;
3. case: lowercase first, `apfel` before `Apfel`;
4. the bytes, so only identical strings are equal. A unique index on a collated column is as strict as one without: `Café` and `cafe` are different values.

A query can sort by another language's rules, `ORDER BY name COLLATE fr`, for a reader in another locale. That query sorts without the index.

## Searching: `lower`, `upper`, and `LIKE`

Punch replaces SQLite's `lower`, `upper`, and `LIKE` on every connection with versions that know every script:

| | SQLite | Punch |
|---|---|---|
| `lower('ÄRGER')` | `ÄRGER` | `ärger` |
| `upper('straße')` | `STRAßE` | `STRASSE` |
| `'Ärger' LIKE 'ä%'` | false | true |

`LIKE` ignores case but not accents: `'Ärger' LIKE 'a%'` is false. The reference app's customer search uses it for a name prefix:

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

The prefix is a `Punch::LikeText`, so a `%` or `_` the user types matches itself instead of acting as a wildcard ([Keeping SQL Safe and Fast](keeping-sql-safe-and-fast.md)). The reference app's spec shows what the search finds:

```crystal source=reference/spec/billing/customers/search_spec.cr select="customers API finds customers by a name prefix in any case and script, sorted as German readers expect"
# spec/billing/customers/search_spec.cr
describe "customers API" do
  it "finds customers by a name prefix in any case and script, sorted as German readers expect" do
    client = AccountsSpec.client
    ["zulu", "Ärger AG", "Anna", "ärmel", "Österreich", "50% Rabatt", "500 Euro"].each_with_index do |name, index|
      client.post(Billing::Customers::Create.path, json: {name: name, email: "n#{index}@example.org"})
    end
    names = ->(q : String) do
      client.get(Billing::Customers::Search.path, query: {q: q}).json["customers"].as_a.map(&.["name"].as_s)
    end
    names.call("ä").should eq(["Ärger AG", "ärmel"])
    names.call("A").should eq(["Anna"])
    names.call("öst").should eq(["Österreich"])
    names.call("50%").should eq(["50% Rabatt"])
    client.get(Billing::Customers::Index.path).json["customers"].as_a.size.should eq(7)
    response = client.get(Billing::Customers::Search.path, query: {q: "a"})
    response.trace.should have_span("Billing::Customers::Queries.search")
  end
end
```

"ä" finds both spellings of the initial, "A" finds only "Anna", since accents count, and "50%" finds the literal percent sign. The results come sorted by the column's collation.

## When a collation's rules change

A collation's rules can change in a Punch release: new rules, a newer Unicode, another provider. An index built with the old rules would then be out of order, and lookups and uniqueness would go wrong without any error. So a collation never changes silently:

- Each collation has a version, such as `punch-latin-1/unicode-17.0.0`, and `punch db:migrate` records the versions the schema uses.
- When a release changes one, the app refuses to open the database, naming the collations.
- The fix is a migration that rebuilds their indexes:

```sql illustrative="the migration a changed collation asks for"
REINDEX de;
```

`punch db:migrate` refuses to run until that migration is pending, then applies it and records the new version. Rebuilding the indexes reads every row of the collated columns, so it is a scan ([Migrations](migrations.md#what-a-migration-costs)).

## Changing a column's collation

SQLite cannot change a column's collation in place. The migration rebuilds the table: create the new table with the collation, copy the rows, drop the old one, and rename the new one. It holds the write lock while it copies, so acknowledge it with `-- rebuild: <table>` ([Migrations](migrations.md#what-a-migration-costs)). Choose the collation when you create the table.

## Tools that cannot read collated columns

The system's `sqlite3`, reporting tools, and anything else that is not a Punch binary do not know Punch's collations. Sorting by, comparing, or writing a collated column there fails with `no such collation sequence`. Use `punch db:shell`, which has them ([SQLite in Punch](sqlite-in-punch.md#inspecting-a-database)).

## Common mistakes

**Sorting in Crystal.** Reading every row to `sort_by(&.name)` is slow and sorts by bytes. Declare `COLLATE` on the column and `ORDER BY` it.

**`lower(name) = lower(:name)` for a case-insensitive lookup.** It cannot use an index on `name`. For a unique, case-insensitive value such as an email address, store it normalized: `Punch::Email` is lowercased when parsed.

**Passing the user's text as a `LIKE` pattern.** A search for "50%" then matches "50 Euro" too. Pass it as a `Punch::LikeText`, with `ESCAPE '\'` in the SQL.

**Expecting "a" to find "Ärger".** Accents count in `LIKE`. Accent-insensitive search is not built yet.

**Using the system's `sqlite3`.** It cannot read collated columns. Use `punch db:shell`.

## Limits

- The collations are for languages written in Latin letters where accented letters are not letters of their own. They are wrong for Swedish, Danish, Norwegian, Polish, and Czech.
- Accent-insensitive matching is not built.
- Punch's `LIKE` replaces SQLite's, which gives up SQLite's index use for prefix patterns: a prefix search reads the rows its other conditions select.

## Rules

- Declare a column's sort order with `COLLATE` in the migration that creates it; never sort in Crystal what the database can sort. ([Sorting: a collation in the schema](#sorting-a-collation-in-the-schema))
- Let text in through primitives, so it is normalized. ([One text, one form](#one-text-one-form))
- Pass user text to `LIKE` as a `Punch::LikeText`, with `ESCAPE '\'`. ([Searching: `lower`, `upper`, and `LIKE`](#searching-lower-upper-and-like))
- When a collation's version changes, write the `REINDEX` migration it asks for. ([When a collation's rules change](#when-a-collations-rules-change))
- Inspect collated columns with `punch db:shell`, never the system's `sqlite3`. ([Tools that cannot read collated columns](#tools-that-cannot-read-collated-columns))

## Reference

- [Text](../text.md): the collations' rules, costs, versions, functions, and custom collations.
- [Queries: filtering by text](../queries.md#filtering-by-text): `Punch::LikeText` and declared search clauses.
