# Deploying

Applies to Punch 0.1.

A Punch app ships as one static binary. The code, the migrations, the assets, the compiled-in credentials file, and the OpenAPI document are inside it, and it runs on one machine, with its SQLite files on local disk, behind a reverse proxy that terminates TLS. A deploy starts the new binary beside the old one, waits until it answers `/up`, moves traffic to it, and stops the old one after its last requests. No request is dropped, and no deploy step runs separately from the binary.

After reading this guide, you will know:

- how the binary and its image are built;
- how a deploy hands over from one process to the next without dropping requests;
- how to deploy with Kamal, or with Caddy and systemd;
- what to check after a deploy, and how to roll back.

Before you start: you know how migrations reach production ([Migrations](migrations.md#migrations-in-production)) and which settings a deployment sets ([Configuring Punch Applications](configuring-punch-applications.md)). `punch new` writes either recipe into an app, with `--deploy kamal` (the default) or `--deploy systemd`; the reference app has both, at `reference/config/deploy.yml` and `reference/deploy/`.

## One binary, one image

The `Dockerfile` that `punch new` writes builds on Alpine Linux against musl, with `--release --static`, so the binary runs on any Linux of the same architecture without libraries. It pins the compiler image and the Litestream release, and checks each download against its checksum.

```console
$ docker build -t examples .                        # the image
$ docker build --target binary --output out .        # only the binary, for systemd
```

While Punch has no public repository, an app depends on a checkout of it by path, and the build takes that checkout as a second context, `--build-context punch=PATH`; the app's README names the exact command.

The image is `FROM scratch`: the binary, Litestream, CA certificates, and time zones, nothing else. There is no shell and no package manager to patch. It runs `serve` as user 1000, with `PUNCH_ENV=production`, `PUNCH_STORAGE=/data`, and `PUNCH_HOST=0.0.0.0` on port 3000. `/data` is a volume, and it must be on the machine's local disk ([SQLite in Punch](sqlite-in-punch.md#local-disk-only)). On the reference app, the image is about 29 MB.

## How a deploy hands over

For a few seconds, two processes run against the same files:

1. **The new process boots.** It applies pending migrations, then opens the databases. Until it is ready, `/up` does not answer 200. A migration that fails stops the boot, so the new process never gets traffic and the old one keeps serving.
2. **The proxy switches.** Once the new process's `/up` answers 200, the proxy sends new requests to it.
3. **The old process drains.** On SIGTERM, its `/up` turns 503 and its responses carry `Connection: close`. After `PUNCH_DRAIN_DELAY_SECONDS`, its listener closes. Requests in progress finish, for up to `PUNCH_SHUTDOWN_TIMEOUT_SECONDS` (25 by default). Then its jobs finish or go back to the queue, the outbox relay stops, its last records reach `ops.db`, the databases close, and Litestream syncs the last writes.

While both run, they share what is on disk and keep what is in memory to themselves:

| | During the overlap |
|---|---|
| database files | shared through SQLite's locks; each process waits for the other's write lock |
| jobs | both claim, each job once; each only payload versions it reads |
| recurring jobs | each run enqueued once, by its key |
| Litestream | runs in the process holding its lock; the new one takes over once the old one has synced |
| live pages | streams end when the old process drains; browsers reconnect to the new one |
| memory caches and memory rate limits | per process; the new one starts empty |

So a migration must leave the previous release working: the old process serves against the new schema until it stops ([Migrations](migrations.md#the-previous-release-keeps-working)).

## With Kamal

Kamal deploys the image to a host behind kamal-proxy. On `kamal deploy`, the new container boots and migrates, kamal-proxy polls its `/up` until it answers 200, switches traffic, waits for requests in progress on the old container, and Kamal stops the old container. The reference app's configuration:

```yaml source=reference/config/deploy.yml
# config/deploy.yml
# Kamal 2 (lib/punch/docs/deployment.md): the image from ../Dockerfile on one host,
# behind kamal-proxy, which terminates TLS, waits for the new container's
# /up, switches traffic, and drains the old container before Kamal stops it.
# Both containers share the volume during the switch: two processes on the
# same SQLite files, as Punch expects.
service: invoicing
image: example/invoicing

servers:
  web:
    hosts:
      - 192.0.2.10
  # A separate jobs process, optional; then set PUNCH_JOBS=off for web.
  # Events its jobs commit (a payment from the webhook) then no longer
  # reach pages open in browsers, which update on the next load.
  # jobs:
  #   hosts:
  #     - 192.0.2.10
  #   cmd: jobs

proxy:
  ssl: true
  host: invoices.example.org
  app_port: 3000
  # With TLS on, kamal-proxy only forwards X-Forwarded-For when asked; the
  # app trusts it from the kamal network (PUNCH_TRUSTED_PROXIES below).
  forward_headers: true
  healthcheck:
    path: /up
    interval: 1
    timeout: 5
  response_timeout: 30

# Kamal waits this long for requests on the old container, then stops it
# with SIGTERM; the app drains for up to PUNCH_SHUTDOWN_TIMEOUT_SECONDS.
drain_timeout: 30
# Pending migrations run at boot, before /up turns 200.
deploy_timeout: 120

registry:
  server: ghcr.io
  username: example
  password:
    - KAMAL_REGISTRY_PASSWORD

# Punch is a path dependency of the reference app, so the build context is
# Punch's root; an app with Punch from Git builds from its own directory.
builder:
  arch: amd64
  context: ..
  dockerfile: Dockerfile

volumes:
  - invoicing_data:/data

env:
  clear:
    PUNCH_PUBLIC_URL: https://invoices.example.org
    # The subnet of Docker's `kamal` network (docker network inspect kamal).
    PUNCH_TRUSTED_PROXIES: 172.18.0.0/16
    PUNCH_BACKUP_URL: s3://example-backups/invoicing
  secret:
    # The credentials file's key, or the credentials themselves as
    # variables (SECRET_KEY_BASE, ISSUER__NAME, MAIL__...).
    - PUNCH_MASTER_KEY
    - AWS_ACCESS_KEY_ID
    - AWS_SECRET_ACCESS_KEY
```

- `volumes` gives both containers the same `/data` during the switch.
- `forward_headers: true` and `PUNCH_TRUSTED_PROXIES` let the app see client addresses through kamal-proxy.
- `deploy_timeout` covers migrations at boot.
- kamal-proxy has switched before the old container hears of the deploy, so `PUNCH_DRAIN_DELAY_SECONDS` stays 0.

The reference app's build context is Punch's root, because it takes Punch by path from inside Punch's repository; an app's own configuration builds from its own directory.

## With Caddy and systemd

On a VM without containers, Caddy terminates TLS and proxies to two local ports, and systemd runs one instance of the app per port. A deploy script alternates between them. The reference app's Caddy configuration:

```text source=reference/deploy/Caddyfile
# Caddy in front of the app on one machine, with systemd (lib/punch/docs/deployment.md).
# Caddy terminates TLS and speaks HTTP/2 to browsers; the app listens on
# two local ports, one at a time except during a deploy (deploy.sh). Caddy
# sends everything to the first healthy one, and retries a request whose
# connection could not be made on the other.
invoicing.example.org {
	encode zstd gzip

	reverse_proxy 127.0.0.1:3000 127.0.0.1:3001 {
		lb_policy first
		# A refused connection (a process gone or not yet listening) is
		# tried again on the other port for up to 10 s.
		lb_try_duration 10s
		lb_try_interval 100ms

		# /up turns 503 the moment a process starts to drain. It keeps
		# accepting for PUNCH_DRAIN_DELAY_SECONDS (3 in the unit), so the
		# next check, within a second, takes it out of rotation first.
		health_uri /up
		health_interval 1s
		health_timeout 2s
		health_fails 1
		health_passes 1
	}
}
```

`deploy/deploy.sh` installs a new binary as a release, starts the idle instance, waits for its `/up`, and then stops the other. If the new one does not become healthy within 60 seconds, it is stopped instead, and the old one keeps running:

```console
$ deploy/deploy.sh out/examples
```

The systemd unit sets `PUNCH_DRAIN_DELAY_SECONDS=3`, so a stopping instance keeps accepting until Caddy's next health check has taken it out of rotation, and `KillMode=mixed`, so only the app gets SIGTERM and it stops Litestream after its last writes. Settings and the master key go in `/etc/<app>/env`, readable only by root.

## After a deploy

- `/up` answers 200, and `bin/<app> config` shows the settings you expect.
- The ops console's overview at `/_punch` shows no new error group and no failing jobs ([Ops data](../ops.md)).
- `bin/<app> backup` shows every file replicating, with a small lag ([Backup and Restore](backup-and-restore.md#is-it-working)).
- `bin/<app> db:status` lists every migration as `applied`.

**Rolling back** is deploying the previous release's binary: `kamal rollback`, or `deploy.sh` with the old binary. The old binary runs against migrations it does not know, which `db:status` lists as `unknown` and which do not stop it, as long as every migration kept the previous release working.

## Measured

The reference app's scripts in `reference/bench/` deploy under load. With kamal-proxy and with Caddy, three handovers under mixed load of 32 connections lost no requests, of about 400,000 each. Without a proxy, a restart cannot avoid a gap, and a request sent on a keep-alive connection just as the drain closes it is lost: run behind a proxy ([Deployment: measured](../deployment.md#measured)).

## Common mistakes

**A migration that breaks the running release.** The old process fails until it stops, and a rollback fails entirely. Add first, remove a release later.

**Storage in the container instead of a volume.** It goes with the container. Mount a volume on local disk at `/data`.

**A shutdown timeout above the supervisor's.** systemd's `TimeoutStopSec` or Kamal's `drain_timeout` kills the process before it has drained. Keep `PUNCH_SHUTDOWN_TIMEOUT_SECONDS` below it, as the recipes do.

**Running without a proxy.** Every restart drops connections. Use kamal-proxy or Caddy.

**Copying the database to deploy.** The data stays on the machine; only the binary changes. To move to a new machine, restore from the backup ([Backup and Restore](backup-and-restore.md#restoring-after-losing-the-machine)).

## Limits

- One machine: there is no load balancing across machines and no standby.
- Linux only for production binaries.
- During a deploy, memory caches and memory rate limits start empty in the new process.

## Rules

- Deploy the binary the recipe builds, behind kamal-proxy or Caddy. ([One binary, one image](#one-binary-one-image), [How a deploy hands over](#how-a-deploy-hands-over))
- Keep the storage on a volume on local disk. ([One binary, one image](#one-binary-one-image))
- Let migrations run at boot, and keep every migration compatible with the previous release. ([How a deploy hands over](#how-a-deploy-hands-over))
- Keep `PUNCH_SHUTDOWN_TIMEOUT_SECONDS` below the supervisor's limit, and set `PUNCH_DRAIN_DELAY_SECONDS` above a polling proxy's health-check interval. ([With Caddy and systemd](#with-caddy-and-systemd))
- After each deploy, check `/up`, the ops console, and the backup. ([After a deploy](#after-a-deploy))

## Reference

- [Deployment](../deployment.md): the binary, migrations at boot, settings, the handover, shutdown, Kamal, Caddy and systemd, and the measurements.
- [HTTP kernel](../http.md): health checks and the drain.
- [Backup](../backup.md): the sidecar during a deploy.
