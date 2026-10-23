# Configuring Punch Applications

Applies to Punch 0.1.

A Punch app has two kinds of configuration. **Settings** configure Punch itself: where the databases are, which port to listen on, which proxy to trust, how much to trace. They are `PUNCH_*` environment variables with defaults. **Credentials** are what the app declares it needs: API keys, its mail server, the company name on its invoices. Both are checked when the app starts, and every problem is reported at once, so a misconfigured deploy fails at boot instead of in the first request that needs the missing value.

After reading this guide, you will know:

- where each kind of configuration lives;
- which settings matter in production;
- how a bad setting is reported, and how to see what a process uses.

Before you start: you know how credentials are declared ([Credentials and Keys](credentials-and-keys.md)).

## Settings and credentials

| | Settings | Credentials |
|---|---|---|
| Configure | Punch: storage, network, threads, tracing, jobs, retention | the app: API keys, its mail server, its own values |
| Declared by | Punch | the app's `Punch::Credentials` type |
| Come from | `PUNCH_*` variables, with defaults | the environment, then the encrypted `config/credentials/<env>.yml.enc` |
| Shown by | `bin/<app> config` | `bin/<app> creds:check` |

A rule of thumb: if Punch would need it in any app, it is a setting; if only this app needs it, it is a credential. A secret is always a credential, never a setting.

## Seeing what a process uses

`config` shows every setting a process would use, checked as `serve` checks them. For the examples app with a storage directory:

```console
$ PUNCH_STORAGE=/var/lib/examples bin/examples config
environment      development
storage          /var/lib/examples (primary: /var/lib/examples/primary.db)
host             127.0.0.1
port             3000
public URL       http://127.0.0.1:3000
trusted proxies  none
trusted origins  none
shutdown         0 s accepting after /up turns 503, then up to 25 s for requests in progress
migrate          on: pending migrations are applied at boot
threads          8
trace            json, spans of 100 % of traces when exported
otlp             off (OTEL_EXPORTER_OTLP_ENDPOINT unset)
backup           off (PUNCH_BACKUP_URL unset)
cache            on, cache.db up to 256 MiB
jobs             on in serve, 10 at once on 2 thread(s), queues: all, kept 72 h
ops              on: ops.db keeps 100 % of successful job runs and events, 14 days; slow requests from 500 ms
idempotency      answers under a key kept 24 h
webhooks         finished deliveries kept 30 days
```

`punch config` in an app's checkout shows the same, and `--json` gives it as data.

## A bad setting stops the boot

Every setting is checked before anything starts, and every problem is reported together:

```console
$ PUNCH_PORT=abc PUNCH_TRACE=loud bin/examples config
examples: The environment has invalid settings:
  PUNCH_PORT must be a port number from 1 to 65535, got "abc"
  PUNCH_TRACE must be json, console, or off, got "loud"
```

Some defaults hold only in development and test. In any other environment, a setting the app cannot do without is required:

```console
$ PUNCH_ENV=production bin/examples config
examples: The environment has invalid settings:
  PUNCH_PUBLIC_URL must be set in production: where people reach the app, such as https://invoices.example.org (links in mail)
```

`serve` and `jobs` refuse to start with the same messages. A process that starts has a configuration that parsed.

## The settings that matter in production

| Setting | Set it to |
|---|---|
| `PUNCH_ENV` | `production`; selects the credentials file and the stricter checks |
| `PUNCH_STORAGE` | a directory on the machine's local disk, such as `/var/lib/<app>` or a container volume |
| `PUNCH_HOST`, `PUNCH_PORT` | `0.0.0.0` in a container; a port behind the proxy |
| `PUNCH_PUBLIC_URL` | the HTTPS address people use; the base of links in mail |
| `PUNCH_TRUSTED_PROXIES` | exactly the proxy in front of the app, so client addresses and HTTPS come from its headers |
| `PUNCH_MASTER_KEY` | the key of the production credentials, from a secret store |
| `PUNCH_BACKUP_URL` and the provider's keys | the replica; without it, `/up` fails outside development ([Backup and Restore](backup-and-restore.md)) |
| `PUNCH_DRAIN_DELAY_SECONDS`, `PUNCH_SHUTDOWN_TIMEOUT_SECONDS` | what the proxy and supervisor need for deploys without dropped requests ([Deploying](deploying.md)) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | an OpenTelemetry collector, if you have one ([Tracing and Metrics](tracing-and-metrics.md)) |

The reference app's Kamal configuration sets them for its container:

```yaml source=reference/config/deploy.yml select=env
# config/deploy.yml
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

Most other settings, such as threads, job concurrency, and retention, have defaults that suit one machine. Change one when a measurement says so; the ops console shows where requests and jobs spend their time ([Ops data](../ops.md)).

The full list, with every default, is in [Tooling: settings](../tooling.md#settings-punch-config).

## Common mistakes

**A secret as a `PUNCH_*` setting or a plain variable the app reads.** Settings are Punch's, and `ENV` reads are reported. Declare the secret as a credential.

**Storage on a network share or a container's writable layer.** Punch refuses the first; the second loses data with the container. Use local disk or a volume on it.

**`PUNCH_TRUSTED_PROXIES` too wide.** Trusting a whole network lets any machine in it claim to be any client. Name the proxy's address or its network; `0.0.0.0/0` is refused.

**Changing a default without measuring.** More job threads or a higher concurrency can slow requests down. Look at the ops console's performance view first.

## Rules

- Configure Punch with `PUNCH_*` settings; declare everything the app itself needs as credentials. ([Settings and credentials](#settings-and-credentials))
- Check a deployment's settings with `config` and its credentials with `creds:check` before it serves. ([Seeing what a process uses](#seeing-what-a-process-uses))
- In production, set at least `PUNCH_ENV`, `PUNCH_STORAGE` on local disk, `PUNCH_PUBLIC_URL`, `PUNCH_TRUSTED_PROXIES`, `PUNCH_MASTER_KEY`, and the backup. ([The settings that matter in production](#the-settings-that-matter-in-production))

## Reference

- [Tooling: settings](../tooling.md#settings-punch-config): every setting and its default.
- [Deployment: settings for deploys](../deployment.md#settings-for-deploys).
- [Credentials](../credentials.md).
