# Backup and Restore

Applies to Punch 0.1.

A Punch app keeps its data in SQLite files on one machine. If that machine's disk is lost, so is the data, unless it was copied somewhere else first. Punch copies every change to S3-compatible storage within about a second, with Litestream running beside the app. Every night it proves the copy can be restored. When you need it, `restore` brings the files back and holds background work until you have checked what the restore changed.

After reading this guide, you will know:

- how to turn backups on, and why a production app without them fails its health check;
- how to tell whether replication works right now;
- how to prove the backup can be restored;
- how to restore after losing the machine, or after a bad change, step by step;
- what recovery mode holds back, and how to end it.

Before you start: the app is deployed with one of the recipes `punch new` writes ([Deployment](../deployment.md)). The Kamal recipe's image includes Litestream; with the systemd recipe, install the `litestream` binary on the machine, or name it in `PUNCH_LITESTREAM`. The examples below come from a drill with the guides' examples app (`docs/guides/examples/`) and a `file://` replica, so you can repeat it without a bucket.

## What is backed up

The storage directory (`PUNCH_STORAGE`) holds four SQLite files. Three are replicated, each to its own path under the backup URL:

| File | Holds | Replicated |
|---|---|---|
| `primary.db` | The app's data | Yes |
| `queue.db` | Jobs waiting, running, and recently finished | Yes |
| `ops.db` | Errors, job runs, and performance for the ops console | Yes, unless exempted |
| `cache.db` | Values that can be computed again | No |

Litestream copies each change from SQLite's write-ahead log about a second after it is written, and a full snapshot once a day. Punch writes Litestream's configuration at every start, runs it as a sidecar of `serve` or `jobs`, and restarts it when it exits. You never configure or start Litestream yourself, and `litestream.yml` in the storage directory is rewritten at every start, so editing it changes nothing.

When several processes share a storage directory, such as `serve` and `jobs`, or the old and the new process during a deploy, exactly one of them runs Litestream. Two Litestreams writing one replica would corrupt it.

## Turning backups on

Set the backup URL and the storage provider's keys in the app's environment:

```sh illustrative="the settings a deploy sets; values depend on your bucket"
PUNCH_BACKUP_URL="s3://backups/examples"   # primary.db goes to backups/examples/primary, and so on
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
```

Providers other than AWS take Litestream's options in the URL's query, such as `?endpoint=https://<account>.r2.cloudflarestorage.com&region=auto`. Keys never go in the URL.

Give every app and every environment its own path. Two apps replicating to the same path destroy each other's backups.

**Without a backup, a production app is unhealthy.** Outside development and test, a database file without a replica makes `/up` answer 503, naming the file, so a deploy without backup settings does not go live unnoticed. To run without a replica on purpose, list the file in `PUNCH_BACKUP_EXEMPT`. Exempting `ops` is reasonable for an app whose ops data is not worth the uploads; exempting `primary` almost never is.

**The bucket must encrypt what it stores.** Litestream does not encrypt replicas, so Punch asks the bucket for its default encryption at start and every six hours, and `/up` fails when there is none. AWS encrypts every bucket by default. A provider that encrypts everything but does not answer that request needs `PUNCH_BACKUP_ENCRYPTION=provider`, which states that you checked. A `file://` replica is not checked.

If Litestream cannot run, because the binary is missing or the storage directory's path is too long for its socket, `serve` and `jobs` refuse to start.

## Is it working?

`backup` shows the state of replication now. Run it with the app's binary (`/opt/<app>/current/<app>` in the systemd recipe), or as `punch backup` in a checkout of the app:

```console
$ bin/examples backup
replica     file:///var/backups/examples
replicated  primary.db, queue.db, ops.db
litestream  0.5.17, process 365666, up 5 s (/var/lib/examples/litestream.sock)
  primary  replicating, lag 0.0 s, last upload 2026-10-23T13:49:23Z
  queue    replicating, lag 0.0 s, last upload 2026-10-23T13:49:23Z
  ops      replicating, lag 0.0 s, last upload 2026-10-23T13:49:23Z
encryption  not checked: file replica
verified    never: the restore check runs every night at 04:00 UTC (punch backup:verify runs it now)
```

The lag of a file is how long its oldest change not yet uploaded has waited: zero when everything is uploaded. Each process measures it every 15 seconds and exports it as the metric `punch.backup.lag`. A lag above five minutes is reported once as a `Punch::Backup::Lagging` error in the ops console.

A file that nobody writes to would have nothing to upload, and from the replica alone it would look like a file whose uploads had stopped. So Punch writes a heartbeat row into each replicated file that has been quiet for 15 seconds. Every replica is then at most about 20 seconds old while Litestream runs.

**The lag does not fail the health check.** A failing `/up` takes the app out of its proxy, and an outage at the storage provider should not take the app down with it. Alert on the lag instead.

## Proving the backup restores

A backup that cannot be restored is not a backup. Every night at 04:00 UTC, the recurring job `Punch::Backup::Verify` restores every replicated file into a scratch directory beside the live files, runs SQLite's integrity check on each, and works out what a real restore would have to reconcile, without changing anything. `backup:verify` runs the same check now:

```console
$ bin/examples backup:verify
restore check passed
  primary.db: transaction 1 of 2026-10-23T13:49:19Z, 159744 bytes in 0.0 s, integrity ok
  queue.db: transaction 2 of 2026-10-23T13:49:28Z, 57344 bytes in 0.0 s, integrity ok
  ops.db: transaction 2 of 2026-10-23T13:49:28Z, 102400 bytes in 0.0 s, integrity ok
  reconciliation (dry run):
    2 to run again (finished at or after the primary's point): Accounts::Emails::NotifyAdded(Accounts::EmailAdded) (job 1, done), Accounts::Emails::SendVerification(Accounts::EmailAdded) (job 2, done)
```

The dry run found two mail jobs that finished at or after the last point of `primary.db` in the replica. Restored at this moment, the queue would remember them as done while the data they followed from might be missing, so a restore proposes to run them again. While the app runs, the files reach the replica within a second or two of each other, so the dry run usually finds nothing, or a few jobs from the last seconds, like these.

`backup` shows the last result, and the metric `punch.backup.verification.age` is the time since a check last passed. Alert when it exceeds about two days: that catches a check that fails and a check that never runs. A failed check is also a `Punch::Backup::VerificationFailed` error in the ops console. Like the lag, it does not fail the health check.

The scratch copies need as much space as the live files, and the check refuses to start when they would leave less than 1 GB, or a tenth of the filesystem, free. With an S3 replica, the check downloads every file every night: with large databases, mind the provider's transfer prices.

## Restoring after losing the machine

This is the runbook. The same steps restore onto the old machine after a bad change, with `--at` in step 3 ([Restoring to a point in time](#restoring-to-a-point-in-time)).

**1. Stop the app.** A restore replaces the database files, so no process may have them open. On the old machine, stop every `serve` and `jobs` process. A restore refuses while one runs:

```text
A process has the files in /var/lib/examples open: stop serve and jobs before restoring.
```

**2. Prepare the machine.** On a new machine, install the release as for a deploy, with the same settings: above all `PUNCH_BACKUP_URL`, the provider's keys, and `PUNCH_STORAGE`. An empty storage directory is fine.

**3. See what would be restored.** `--dry-run` downloads nothing:

```console
$ bin/examples restore --dry-run
would restore primary.db, queue.db, ops.db, each to its latest point (dry run, nothing downloaded)
  primary.db: transaction 4 of 2026-10-23T13:49:59Z, 4 replica files, 15975 bytes
  queue.db: transaction 6 of 2026-10-23T13:49:59Z, 6 replica files, 7622 bytes
  ops.db: transaction 5 of 2026-10-23T13:49:59Z, 5 replica files, 19805 bytes
```

Check the times. If one file's replica ends well before the others, the restore aligns every file to the earliest end, so the files agree with each other, at the cost of the newest writes of the others.

**4. Restore:**

```console
$ bin/examples restore
restored primary.db, queue.db, ops.db, each to its latest point
  primary.db: transaction 4 of 2026-10-23T13:49:59Z
  queue.db: transaction 6 of 2026-10-23T13:49:59Z
  ops.db: transaction 5 of 2026-10-23T13:49:59Z
  reconciliation: nothing to do
recovery mode: requests are served; job workers, the outbox relay, and recurring schedules wait.
Check the lists with `punch recovery`, then confirm with `punch recovery:confirm`.
```

The restore works in a scratch directory and checks every file's integrity before it touches the live ones. When any file cannot be restored intact, nothing is replaced. Files it does replace are moved aside into `replaced-<time>-<random>/` in the storage directory, never deleted, together with `cache.db`, whose values came from the replaced data.

**5. Start the app.** It serves requests in recovery mode: job workers, the outbox relay, and the recurring schedules wait. `/up` is healthy, so the app goes back into its proxy. Commands still run, and the jobs they enqueue wait in the outbox. `backup` shows the state as `recovery since …`.

**6. Review the reconciliation.** `recovery` shows it, and so does the ops console at `/_punch/recovery`:

```console
$ bin/examples recovery
recovery mode since 2026-10-23T13:50:09Z: background work waits for `punch recovery:confirm`
restored primary.db, queue.db, ops.db, each to its latest point
  primary.db: transaction 4 of 2026-10-23T13:49:59Z
  queue.db: transaction 6 of 2026-10-23T13:49:59Z
  ops.db: transaction 5 of 2026-10-23T13:49:59Z
  reconciliation: nothing to do
Confirming hands 0 outbox entries over again, discards 0 jobs, runs 0 finished jobs again, makes 0 interrupted jobs ready, and starts background work. 0 jobs ran for Commands not in the primary: review what they did.
```

This drill stopped the app cleanly before losing the machine, so the files agree and there is nothing to reconcile. After a real loss there usually is; the next section explains each list. Look hardest at jobs that ran for Commands not in the restored data: they did something outside the app, such as sending a mail or charging a card, for writes that the restore took back. Nothing can undo that automatically.

**7. Confirm.** `recovery:confirm`, or the button in the ops console, applies the lists while the app runs, and every process starts its background work within two seconds:

```console
$ bin/examples recovery:confirm
recovery confirmed: 0 outbox entries to hand over again, 0 jobs discarded, 0 to run again, 0 interrupted made ready
Running processes start background work within 2 s.
```

**8. Clean up later.** Once you are sure the restored data is right, remove the `replaced-*` directory from the storage directory.

Replication continues into the same replica after a restore. Litestream notices that the restored file is behind its replica and continues from it, so a later restore of the latest point has the writes made since this restore.

## What recovery mode holds back, and why

`primary.db` and `queue.db` are replicated separately, so after a restore they may stand at slightly different points. A job may be in the queue for an outbox entry the data no longer has, or the data may hold an outbox entry the queue never received. Starting background work blindly would run jobs for writes that are gone, or lose jobs for writes that exist. So a restore compares the two files, and recovery mode waits until a person has seen the result:

| List | What it holds | Confirming does |
|---|---|---|
| hand over again | Outbox entries marked as handed over, with no job in the queue: the queue is older | Hands them over again |
| discard | Unfinished jobs from outbox entries the data does not have: the queue is newer | Discards them |
| ran for a Command not in the primary | Finished jobs from such entries | Nothing: their effects outside the app remain, for you to review |
| run again | Jobs that finished at the data's point, after it, or within 5 seconds before it: their writes may be gone | Makes them ready, with fresh attempts |
| interrupted | Jobs running at the queue's point | Makes them ready again, as after a crash |

Only jobs and entries within the job retention window (72 hours by default) are compared. A job that something else enqueued, such as another job or a schedule, cannot be traced to a Command; it can only appear as run again or interrupted.

## Restoring to a point in time

A bad migration or a mistaken bulk change is replicated like any other write within a second. To go back to before it, restore to a time:

```console
$ bin/examples restore --at 2026-10-14T09:30:00Z
```

Each file is restored to its last transaction at that second or before, then the steps from step 4 on are the same. The time is in RFC 3339 with an offset. Run it with `--dry-run` first to see which transactions it picks. Everything written after that time is not in the restored files; the files moved aside in `replaced-*` still have it, if you need to recover single rows from them.

## Practising the drill

Run the drill before you rely on the backup, and again after changing the storage provider or the deploy. Locally, a `file://` replica needs no account:

```console
$ export PUNCH_STORAGE=/tmp/drill PUNCH_BACKUP_URL=file:///tmp/drill-replica
$ bin/examples db:migrate
$ bin/examples serve
```

Use the app for a minute, then stop it (Ctrl-C), move `/tmp/drill` away to play the lost machine, and follow the runbook from step 3. To try S3 without an account, run a local S3 gateway such as versitygw ([Backup: testing](../backup.md#testing)).

## Costs

- **Requests.** A quiet file costs one upload every 15 to 20 seconds for its heartbeat: about 5,000 requests a day per file, about $0.75 a month per quiet file at AWS S3's request price. Busy files get no extra writes.
- **Latency.** Litestream holds a read transaction between its checkpoints, and its checkpoint briefly takes the write lock. Measured on the reference app at 1,000 requests a second, the 99th percentile went from 1.1 ms to 1.4 ms. Under saturation, the tail grows much more ([Backup: cost](../backup.md#cost)).
- **Transfer.** The nightly check downloads every replicated file.

## Common mistakes

**Editing `litestream.yml`.** Punch rewrites it from the settings at every start. Change `PUNCH_BACKUP_URL` or `PUNCH_BACKUP_EXEMPT` instead; `backup:config` prints what Punch writes.

**One backup path for two apps or environments.** Staging replicating into production's path corrupts both replicas. Give each its own path.

**Setting `PUNCH_BACKUP_ENCRYPTION=provider` to make `/up` pass.** It is a statement that the provider encrypts every object at rest. If the check fails because of a missing permission, grant `s3:GetEncryptionConfiguration`; if the bucket has no default encryption, turn it on.

**Restoring while the app runs.** The restore refuses with the message in step 1. Stop every `serve` and `jobs` process that uses the storage directory, then restore.

**Confirming without reading the lists.** Confirming applies the lists as proposed. Read "ran for a Command not in the primary" first: those effects happened, and the restored data does not know.

**Deleting `replaced-*` at once.** It holds the files as they were before the restore, the only copy of anything written after the restored point.

**Treating a silent check as a passing one.** The nightly check reports failures, but a check that never runs reports nothing. Alert on `punch.backup.verification.age`.

## Limits

- Litestream does not encrypt replicas; the bucket's encryption at rest is what protects them.
- The three files are replicated separately. A restore aligns them to the earliest replica's end and reconciles the queue with the data; it cannot make them agree to the transaction.
- The newest second or so of writes before a loss can be missing from the replica.
- A restore brings back whole files. There is no restore of single tables or rows; take them from the `replaced-*` files by hand.
- Replication is to one replica. There are no read replicas and no standby that takes over by itself.

## Rules

- Set `PUNCH_BACKUP_URL` for every deployed environment, with a path of its own; exempt a file only on purpose. ([Turning backups on](#turning-backups-on))
- Never edit `litestream.yml`; change the settings. ([What is backed up](#what-is-backed-up))
- Use a bucket that encrypts at rest; claim `PUNCH_BACKUP_ENCRYPTION=provider` only when it is true. ([Turning backups on](#turning-backups-on))
- Alert on `punch.backup.lag` and on `punch.backup.verification.age`. ([Is it working?](#is-it-working), [Proving the backup restores](#proving-the-backup-restores))
- Stop every `serve` and `jobs` before `restore`, and run `--dry-run` first. ([Restoring after losing the machine](#restoring-after-losing-the-machine))
- Read the reconciliation before `recovery:confirm`, especially the jobs that ran for Commands not in the restored data. ([What recovery mode holds back, and why](#what-recovery-mode-holds-back-and-why))
- Keep `replaced-*` until the restored data is confirmed good. ([Restoring after losing the machine](#restoring-after-losing-the-machine))
- Run the drill before relying on the backup. ([Practising the drill](#practising-the-drill))

## Reference

- [Backup](../backup.md): every setting, the sidecar's locking and restarts, the heartbeat, the lag, the check's steps, the restore's steps, and the measured costs.
- [Deployment](../deployment.md): the recipes, and how they stop the app so its last writes reach the replica.
- [Ops data](../ops.md): the console that shows errors, the reconciliation, and the confirm button.
- [Metrics](../metrics.md): exporting `punch.backup.*`.
