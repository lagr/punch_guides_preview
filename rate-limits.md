# Rate Limits

Applies to Punch 0.1.

Some operations must not run as often as a client likes: signing in, where each attempt is a password guess; signing up, where each attempt creates an account; sending invoices, where each one mails a customer. In Punch, a limit is an annotation on the operation, counted per key, and answered with 429 and the time to wait. Every Command that runs without a signed-in user must have one, or say why it does not.

After reading this guide, you will know:

- how to protect sign-in and sign-up;
- which key and which store to choose for a limit;
- what a refused client sees;
- why a Command open to anyone does not compile without a limit;
- how to unlock a user, and how to test limits.

Before you start: you know how sign-in works ([Authentication](authentication.md#checking-a-password)).

## Declaring a limit

The accounts slice's sign-in has two limits:

```crystal source=docs/guides/examples/app/accounts/sessions/operations/create.cr select=Accounts::Sessions::Create elide=call
# app/accounts/sessions/operations/create.cr
module Accounts::Sessions
  # Starts a session: signs in with an email address and a password.
  # Punch checks the password before the transaction; the same answer
  # comes for an unknown address and a wrong password (lib/punch/docs/auth.md). Punch
  # publishes `SignInFailed` for a refused one.
  @[Punch::Public]
  @[Punch::RateLimit(10, per: 15.minutes, by: :email, store: :durable)]
  @[Punch::RateLimit(60, per: 1.minute, by: :ip)]
  struct Create < Punch::Command
    struct Input
      include Punch::Input

      getter email : Punch::Email
      getter password : Punch::Auth::Password
    end

    alias Output = SignedIn | Punch::Auth::InvalidCredentials

    def call(input : Input, db : Punch::Writer, auth : Punch::Auth::Authentication, password : Punch::Auth::PasswordCheck,
             events : Punch::Events(Punch::Auth::SignInSucceeded | Punch::Auth::SignInFailed)) : Output
      # ...
    end
  end
end
```

`@[Punch::RateLimit(10, per: 15.minutes, by: :email, store: :durable)]` allows ten attempts at one address in 15 minutes. A key may use its whole limit at once, then gets one more request every period divided by the limit, here one every 90 seconds. There is no burst at the edge of a window, because there are no windows.

Several limits on one operation are all checked, in order, and the first that refuses answers.

## Choosing the key

| `by:` | Counts per | Checked |
|---|---|---|
| `:ip` | client address; IPv6 per /64 | before the body is read |
| a field, such as `:email` | the field's value in the input | after the input parsed |
| `:actor` | signed-in user | inside the transaction, after authentication |
| `:token` | API token | inside the transaction, after authentication |

Combine them. A sign-in limited only by address can be guessed from many addresses at once, one limited only by email address lets one client try every account. With both, an attacker gets neither.

Behind a proxy, the client address comes from the forwarded header, and only when the proxy is listed in `PUNCH_TRUSTED_PROXIES`. Never build a key from a header yourself: any client can send any header, and a limit keyed by one is a limit every client chooses for itself.

## Choosing the store

| `store:` | Kept | For |
|---|---|---|
| `:memory` (the default) | in the process | throttles: per address, per user, per token |
| `:durable` | in `cache.db`, one small write per check | security limits: sign-in, sign-up, password reset, token creation |

A durable limit survives a restart and is shared by every process on the machine, so a deploy does not hand an attacker a fresh set of attempts. Memory limits are cheaper and reset with the process.

## What a refused client sees

A refused request gets 429 with `Retry-After` and Problem Details naming the limit:

```json illustrative="the shape of the answer"
{"type": "about:blank", "title": "Too Many Requests", "status": 429, "code": "rate_limited",
 "retry_after": 12, "limit": "accounts.sessions.create.ip", "request_id": "…"}
```

A browser gets Punch's problem page instead. Every response of a limited operation carries `RateLimit-Policy` and `RateLimit` headers, so a client can slow down before it is refused, and `openapi.json` documents the 429. On an agent-readable page, each action states its limits ([Views and pages](../views.md#pages-agents-read)).

## Commands open to anyone are limited

A Command that runs without a signed-in user, because it has no actor or an optional one, must have a limit. Otherwise it does not compile:

```crystal fails=spec/fail/command_without_rate_limit.cr
module Notes
  @[Punch::Public]
  struct Create < Punch::Command
    struct Input
      include Punch::Input
    end

    def call(input : Input, db : Punch::Writer) : Nil
    end
  end
end

Punch::Routing::Router.new(Punch::Database.memory)
```

```text
Error: Notes::Create is a Command that runs without a signed-in user, so it needs a rate limit, @[Punch::RateLimit(5, per: 1.minute, by: :ip)], or @[Punch::RateLimitExempt("reason")] (lib/punch/docs/rate-limits.md).
```

A Command that must stay open says why, with a reason that is true:

```crystal illustrative="an exemption with its reason"
@[Punch::RateLimitExempt("signed by the payment provider, which retries")]
```

Commands with a signed-in user are not forced to have a limit, but give one to anything that costs: the reference app's `Send` allows each user 30 invoices an hour, since each mails a customer.

## Unlocking and inspecting

`bin/<app> limits` lists every operation's limits and exemptions, and for durable limits the keys held now. When a user has locked themselves out of signing in, `bin/<app> limits:reset accounts.sessions.create.email ada@example.org` lets that key start over. Memory limits reset with the server.

Outside operations, such as a job calling another system that allows ten requests a second, `Punch::RateLimits::Limiter` is the same counter as a plain object ([Rate limits: outside operations](../rate-limits.md#outside-operations)).

## Testing limits

Every example starts with every key at its full limit. `Punch::Spec.rate_limit_clock` stops time for limiters, and the `be_rate_limited` matcher checks a refusal. The accounts slice's spec:

```crystal source=docs/guides/examples/spec/accounts/sessions/create_spec.cr select="signing up and in with a browser throttles guessing at one address, whatever the client"
# spec/accounts/sessions/create_spec.cr
# ...

describe "signing up and in with a browser" do
  # ...

  it "throttles guessing at one address, whatever the client" do
    sign_up(Punch::Spec.client)
    10.times do
      Punch::Spec.client.post(Accounts::Sessions::Create.path, json: {email: "ingrid@example.org", password: "guess"}).status.should eq(HTTP::Status::UNPROCESSABLE_ENTITY)
    end
    limited = Punch::Spec.client.post(Accounts::Sessions::Create.path, json: {email: "ingrid@example.org", password: PASSWORD})
    limited.should be_rate_limited("accounts.sessions.create.email")
  end
end
```

## Common mistakes

**Only a limit by address on a sign-in.** Many clients can then guess at one account. Add a durable limit by `:email`.

**A memory limit for a security limit.** A restart or a deploy resets it. Use `store: :durable`.

**An exemption to make the build pass.** The reason is a statement about who can call the Command. Write a limit instead, unless something else, such as a signature, already bounds it.

**A key from a header.** Clients choose their headers. Use `:ip`, which honours only trusted proxies, or a field of the input.

## Limits

- Limits count per machine: memory limits per process, durable limits per `cache.db`.
- Durable limits start over after a restore, since `cache.db` is not backed up.

## Rules

- Limit every Command that runs without a signed-in user, or exempt it with a true reason. ([Commands open to anyone are limited](#commands-open-to-anyone-are-limited))
- Limit sign-in both by client address and, durably, by the address being tried. ([Choosing the key](#choosing-the-key))
- Use `store: :durable` for security limits. ([Choosing the store](#choosing-the-store))
- Never key a limit by a header; set `PUNCH_TRUSTED_PROXIES` behind a proxy. ([Choosing the key](#choosing-the-key))

## Reference

- [Rate limits](../rate-limits.md): every argument, the stores, the algorithm, answers, commands, and specs.
- [HTTP kernel](../http.md): trusted proxies and the client address.
