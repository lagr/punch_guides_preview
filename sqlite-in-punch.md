# SQLite in Punch

Applies to Punch 0.1.

A Punch app keeps its data in SQLite files on the machine that runs it. There is no database server to install, connect to, or keep in step with the app: the database engine is compiled into the app's binary, and a query is a function call rather than a round trip over the network. Punch is built around that. It uses SQLite's strengths, such as cheap small queries, one writer at a time, and several database files, instead of hiding them behind an abstraction that would also fit Postgres.

After reading this guide, you will know:

- which files an app stores, and which of them your code touches;
- how Punch schedules reads and writes, and what that means for a slow transaction;
- why the files must be on local disk;
- how to inspect a database safely;
- how far one machine goes.

Before you start: you know what Queries and Commands are ([Operations and Routing](operations-and-routing.md)).

## The storage directory

Everything an app stores is in one directory, named by `PUNCH_STORAGE`:

| File | Holds | Whose |
|---|---|---|
| `primary.db` | the app's data: its tables, from its migrations, and Punch's tables for sessions, roles, the outbox, idempotency keys, and webhooks | the app's |
| `queue.db` | background jobs | Punch's |
| `ops.db` | errors, job runs, and performance, for the ops console | Punch's |
| `cache.db` | values that can be computed again | Punch's |

Your SQL reads and writes `primary.db` only, through your migrations and named queries. The other three files belong to Punch: their tables are created and upgraded by Punch, and you reach them through its APIs (`jobs.enqueue`, the ops console, `Punch::Cache`) and its commands (`punch jobs:list`, `punch ops`). Splitting them keeps job claims, error records, and cache writes from queuing behind the app's own writes, and lets each file be backed up, or not, on its own ([Backup and Restore](backup-and-restore.md)).

Each file is in SQLite's WAL mode, so a file comes with a `-wal` and a `-shm` file beside it. Copy or move them only with the app stopped, or through a backup.

## Readers and the writer

Each database file has one writer connection and a pool of read-only connections.

- **Reads run in parallel.** A Query receives a `Punch::Reader`, a read transaction on one of the read connections. Every statement in it sees the same snapshot of the database, even while writes commit meanwhile. There is about one reader per thread.
- **Writes run one after another.** A Command receives a `Punch::Writer`, a write transaction on the single writer connection. Write transactions queue in the process and run in turn, each one committed before the next starts. Inside one process, writes never meet a "database is locked" error.

One writer sounds like a bottleneck, and it is the reason Punch is fast: a write transaction of a few small statements takes tens of microseconds, there is no lock contention between writers, and no write ever waits for a network. Measured on the reference app, writes alone run at about 20,000 requests a second on one machine.

The cost is that a write transaction holds every other write. So keep Commands short:

- **Never call another system inside a Command.** An HTTP call of 300 ms inside a write transaction stops every write in the app for 300 ms. Enqueue a job instead, which calls out without holding a transaction ([Background Jobs](background-jobs.md)).
- **Do slow computation before the transaction.** Punch hashes passwords before the Command's transaction opens, for this reason ([Authentication](../auth.md)).
- **Read in the Command what it decides on.** A Command reads inside its own write transaction, so what it reads cannot change before it writes. There is no need, and no way, to lock rows.

Long reads have a cost too: a read transaction keeps the WAL from being checkpointed past its snapshot, so the `-wal` file grows while it runs. The ops console shows how long each route's writes waited for the writer, and the writer's queue and the WAL's size are metrics ([Metrics](../metrics.md)).

## Reading and writing in code

Your code rarely opens a transaction itself. A Query or Command gets its handle as an argument of `call`, and a job opens short ones with `job.read` and `job.write`. Inside, it calls the typed methods `punch db:generate` writes from named SQL, so the reference app holds no hand-written database calls at all ([Queries](../queries.md)).

The handle's type is the guarantee. A `Punch::Reader` has no method that writes:

```crystal fails=spec/fail/query_exec.cr
db = Punch::Database.memory
db.read { |reader| reader.exec("DELETE FROM invoices") }
```

```text
Error: undefined method 'exec' for Punch::Reader
```

When you do write SQL by hand, arguments bind to `?` and are plain values: `Nil`, `Bool`, `Int32`, `Int64`, `Float64`, `String`, or `Bytes`. A domain type does not bind by itself:

```crystal fails=spec/fail/sql_argument_type.cr
db = Punch::Database.memory
email = Punch::Email.parse("ann@example.org").as(Punch::Email)
db.write { |writer| writer.exec("INSERT INTO customers (email) VALUES (?)", email) }
```

```text
Error: SQL argument 1 is Punch::Email; bind Nil, Bool, Int32, Int64, Float64, String, or Bytes. Convert domain types where they are bound, such as `email.to_s` (lib/punch/docs/persistence.md).
```

The generated methods do that conversion for you, from the column types the entity maps in its `columns.cr`. A value never goes into the SQL text itself ([Keeping SQL Safe and Fast](keeping-sql-safe-and-fast.md)).

## The SQLite inside the binary

Punch does not use the system's SQLite. It vendors one release, SQLite 3.53.4, and compiles it into every binary with the same options. So `STRICT` tables, `RETURNING`, JSON functions, full-text search, and collations behave the same on a laptop, in CI, and in production.

Tables are `STRICT`, so a column declared `INTEGER` refuses a string instead of storing it. The tables a named query maps must be `STRICT`, and `punch check` says so when one is not.

## Local disk only

WAL mode needs shared memory and file locks that network and virtual filesystems do not honour reliably. On such a filesystem, a reader can see a snapshot that never existed. So Punch checks the storage directory when it opens a database, and refuses NFS, SMB, 9p, FUSE (which includes virtiofs and sshfs), and similar, with a message that names the directory:

```text
/srv/shared/storage is on fuse, a shared or virtual filesystem where SQLite's WAL snapshots are not reliable. Put the database on local disk (lib/punch/docs/persistence.md).
```

Put `PUNCH_STORAGE` on the machine's own disk. In a container, use a volume on local disk; the Kamal recipe `punch new` writes does ([Deployment](../deployment.md)).

## Inspecting a database

`punch db` shows the SQLite that is compiled in, its options, and Punch's collations. With a path, it also shows the file:

```console
$ punch db /var/lib/examples/primary.db
...
Collations: de en es fr it nl pt (punch-latin-1/unicode-17.0.0)

/var/lib/examples/primary.db
  filesystem: ext4
  size: 163840 bytes, WAL 4152 bytes
  journal_mode: wal
  page_size: 4096
  page_count: 40
  freelist_count: 0
  user_version: 0
  application_id: 0
  auto_vacuum: 0
  encoding: UTF-8
```

`punch db:shell PATH` opens SQLite's own shell on a file, with Punch's collations and text functions registered. Use it instead of the system's `sqlite3`: stock tools do not know the collations, so they cannot sort or write indexed columns that use one ([Text and Sort Order](text-and-sort-order.md)).

The shell is a tool for looking. Changes made in it bypass the app's rules, its events, and its audit trail. Change data through the app's Commands, or through a seed or a one-off script that calls them.

## How far one machine goes

Punch runs on one machine: one storage directory, one writer per file. Several processes on that machine can share the files, such as the old and the new process during a deploy, or `serve` and a separate `jobs` process; they take turns on SQLite's lock.

On the reference app, measured on one machine with 8 shared cores, a mix of four reads to one write ran at about 49,000 requests a second before it had sign-in and organizations, and at about 32,000 with them, where every request also loads its user and their roles. Reads alone ran at about 67,000 a second, writes alone at about 20,000 ([the reference app's README](../../reference/README.md)). Most apps never come near that. The limits that matter first are usually the disk's size and a Command that holds the writer too long; the second shows in the writer's wait time before it hurts.

What one machine does not give you is a standby: if the machine is lost, the app is down until a restore on another machine is done ([Backup and Restore](backup-and-restore.md)).

## Common mistakes

**Calling out inside a Command.** A payment provider's API called in `call` holds every write in the app until it answers. Enqueue a job.

**Opening `queue.db` or `ops.db` from the app's SQL.** Their layout is Punch's and changes between releases. Use `punch jobs:*`, `punch ops`, and the ops console.

**Using the system's `sqlite3` on a collated column.** It fails with `no such collation sequence: de`. Use `punch db:shell`.

**Putting the storage on a network share.** Punch refuses to open it, with the message above. A shared folder of a virtual machine counts too.

**Copying a database file while the app runs.** The copy can miss what is in the WAL. Use `punch backup:verify` or a restore to get a consistent copy ([Backup and Restore](backup-and-restore.md)).

## Limits

- SQLite is the only engine and one machine the only topology. There are no read replicas and no failover.
- One writer per database file: write throughput is what one transaction after another allows.
- Punch's files are not for direct use, and their layout may change with any release.

## Rules

- Read and write only `primary.db`, through migrations and named queries; reach Punch's files through its APIs and commands. ([The storage directory](#the-storage-directory))
- Keep write transactions short: never call another system, sleep, or compute something slow inside a Command. ([Readers and the writer](#readers-and-the-writer))
- Keep `PUNCH_STORAGE` on local disk. ([Local disk only](#local-disk-only))
- Inspect with `punch db` and `punch db:shell`; change data through the app's Commands. ([Inspecting a database](#inspecting-a-database))

## Reference

- [Persistence](../persistence.md): the settings, pragmas, checkpoints, the handle methods, and the driver.
- [Metrics](../metrics.md): the writer's queue, the WAL, and checkpoints.
- [Text](../text.md): collations.
- [Migrations](../migrations.md): how `primary.db` gets its tables.
