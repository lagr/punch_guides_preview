# Authentication

Applies to Punch 0.1.

Authentication answers one question for every request: who is this acting for? Punch has the parts that are easy to get wrong: sessions and tokens that are random and stored hashed, password hashing that never holds up the database, one-time links that survive mail scanners, a second factor, and the cross-site check. The flows built from them, sign-up, sign-in, password reset, and the rest, are the app's own operations, which `punch new` writes into the accounts slice and you change like any other code.

After reading this guide, you will know:

- how a request gets its actor, and how an operation asks for it;
- how sessions, API tokens, and one-time links work, and why none of them is signed;
- how passwords are checked without holding the database's writer;
- how to change a sign-in flow safely, and where the second factor fits;
- how to sign in and use tokens in specs.

Before you start: you know how operations and their `call` parameters work ([Operations and Routing](operations-and-routing.md)). The examples come from the accounts slice of the guides' examples app (`docs/guides/examples/app/accounts/`), which `punch new` wrote with the fewest parts; the reference app's slice has every part.

## The actor

An app declares one type that stands for whoever a request acts for. `punch new` writes it into `app/actor.cr`:

```crystal source=docs/guides/examples/app/actor.cr
# app/actor.cr
module Examples
  # Who a request acts for: a signed-in user, by session or API token
  # (lib/punch/docs/auth.md). Operations that need one take it in `call`; Punch
  # answers 401 when there is none.
  struct Actor
    include Punch::Auth::Actor

    getter user : Accounts::Users::Record
    getter principal : Punch::Auth::Principal

    def initialize(@user, @principal)
    end

    # The language the user chose, which wins over Accept-Language.
    def locale : Punch::LocaleTag?
      user.locale
    end

    # `nil` when the user no longer exists.
    def self.load(principal : Punch::Auth::Principal, db : Punch::Reader | Punch::Writer) : self?
      user = Accounts::Users::Table.find(db, principal.user_id(Accounts::Users::ID)) || return
      new(user, principal)
    end
  end
end
```

An operation asks for the actor by its type in `call`:

| `call` takes | Means |
|---|---|
| `actor : Examples::Actor` | a signed-in user is required; without one, 401 |
| `actor : Examples::Actor?` | anyone; `nil` when nobody is signed in |
| no actor | the operation does not look |

The session or token is looked up, and the actor loaded, inside the operation's own transaction. So a Command sees the user as of the same snapshot it writes in, and a revoked session cannot slip through between the check and the write.

A browser without a session that asks for a page needing one is sent to the sign-in page, once `app/setup.cr` names it, as the template's does. A JSON client gets the 401 with `WWW-Authenticate: Bearer`.

## Sessions and API tokens

A **session** is a row in Punch's table `punch_sessions`. Its ID travels in the cookie `__Host-session`, which is `Secure`, `HttpOnly`, and `SameSite=Lax`, and which browsers refuse from a subdomain. A session ends after 7 days without a request, and 30 days after sign-in whatever happens. Signing in always makes a new session, so a session ID never survives a sign-in.

An **API token** is for programs and agents, sent as `Authorization: Bearer <token>`. It acts as its user, always expires, is shown once when it is made, and records when it was last used. Users make tokens for their agents and scripts on a page of the app; the reference app's `app/accounts/tokens/` is that page, and the tokens part of `punch new` writes it. Making a token needs a recent sign-in, so a token cannot make another token.

Both are random values, 256 bits for a token, and only their SHA-256 digest is stored. A copy of the database holds no usable session or token, and revoking one is deleting its row. They are never signed: a signed token would stay valid until it expires, whatever happened to its user, and would depend on the key ring. A token also carries a checksum, so a mistyped or invented token is refused without a lookup, and secret scanners can recognize a leaked one.

## Checking a password

Hashing a password properly is slow on purpose: PBKDF2 with 600,000 iterations takes about 70 milliseconds. Inside a Command's transaction, that would hold the database's single writer, and every other write in the app would wait. So Punch never hashes inside a transaction. The router does it first, on threads of their own, and hands the outcome to the Command.

The sign-in Command of the accounts slice:

```crystal source=docs/guides/examples/app/accounts/sessions/operations/create.cr
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
      user_id = password.confirm(db, Users::ID) || return Punch::Auth::InvalidCredentials.new
      # No user has a second factor: the app offers none.
      session = auth.sign_in(user_id, method: "password").as(Punch::Auth::Session)
      events.publish(Punch::Auth::SignInSucceeded.new(user_id.value, session.id, "password"))
      SignedIn.new(user_id)
    end
  end
end
```

- `password : Punch::Auth::PasswordCheck` is the outcome of the check. The input must have `email` and `password`, and the Command must be limited per address; otherwise it does not compile.
- `password.confirm(db, Users::ID)` returns the user, after checking inside the transaction that the address is still theirs. An unknown address, an address without a password, and a wrong password all take the same work and give the same answer, 422 `invalid_credentials`, so the answer does not tell whether an address has an account.
- `auth.sign_in` starts the session. Cookie changes reach the response only if the Command commits.
- A refused sign-in rolls back, its events with it, so Punch publishes `Punch::Auth::SignInFailed` in a transaction of its own afterwards.

A new password is hashed the same way. Sign-up takes a `Punch::Auth::NewPassword`, at least 12 characters, and `input.password.hashed` is ready to store when `call` runs:

```crystal source=docs/guides/examples/app/accounts/sign_up/operations/create.cr select=Accounts::SignUp::Create
# app/accounts/sign_up/operations/create.cr
module Accounts::SignUp
  # ...

  # Creates a user with an email identity and a password, and signs them
  # in. The password is hashed before the transaction opens (lib/punch/docs/auth.md).
  # The language the browser asked for becomes the user's. The address is
  # unverified until its owner follows the mailed link (`EmailAdded`).
  @[Punch::Public]
  @[Punch::RateLimit(10, per: 1.hour, by: :ip)]
  struct Create < Punch::Command
    struct Input
      include Punch::Input

      getter name : Punch::NonEmptyString
      getter email : Punch::Email
      getter password : Punch::Auth::NewPassword
    end

    alias Output = SignedUp | Punch::Auth::EmailIdentities::EmailTaken

    def call(input : Input, db : Punch::Writer, auth : Punch::Auth::Authentication, locale : Punch::Locale,
             events : Punch::Events(EmailAdded | Punch::Auth::SignInSucceeded)) : Output
      tag = Punch::LocaleTag.parse(locale.code).as?(Punch::LocaleTag)
      user = Users::Table.insert(db, name: input.name, created_at: Time.utc, locale: tag)
      # A taken address rolls the new user back with it (lib/punch/docs/queries.md).
      identity = Punch::Auth::EmailIdentities.add(db, user.id, input.email)
      return identity if identity.is_a?(Punch::Auth::EmailIdentities::EmailTaken)

      Punch::Auth::Passwords.set(db, user.id, input.password.hashed)
      # A new user has no second factor yet.
      session = auth.sign_in(user.id, method: "password").as(Punch::Auth::Session)
      events.publish(EmailAdded.new(user.id.value, identity.email))
      events.publish(Punch::Auth::SignInSucceeded.new(user.id.value, session.id, "sign_up"))
      SignedUp.new(user.id)
    end
  end
end
```

Password fields print as `[REDACTED]`, do not serialize, and are never written back into a form that is shown again. When the hashing threads are full, the answer is 503 with `Retry-After` at once, instead of a queue that grows under attack.

## Who someone is, and how they prove it

Punch keeps three things apart:

- the **user**, the app's own table: who someone is;
- the **identity**, an email address they are known by, verified or not; a user can have several;
- the **authenticator**, what proves it is them: a password, a mailed link, an authenticator app.

A user without a password is normal, if they sign in by mailed link. Removing an address refuses to take a user's last way to sign in (409 `last_sign_in_method`).

## Links in mail

Password resets, email verification, and sign-in by mail use one-time tokens: random, single use, with a purpose and an expiry, stored hashed like sessions. Mail scanners fetch links before people click them, so the page behind a link only looks its token up and shows a button; the Command behind the button uses it up. A link that was fetched by a scanner still works for the person.

## The second factor

The reference app offers an authenticator app (TOTP) with recovery codes, written by `punch new` unless `--no-second-factor` leaves it out. Once a user has confirmed one, `auth.sign_in` returns `Punch::Auth::SecondFactorRequired` instead of a session, and the browser goes to a second step that asks for the code. Every sign-in Command handles both, a mailed link included. The authenticator's secret is encrypted with the key ring; recovery codes are stored as digests ([Authentication: second factor](../auth.md#second-factor)).

**Recent authentication.** Changes such as adding an address, making a token, or removing the second factor need a session that proved who the user is within the last 15 minutes. An operation asks for it with a policy ([Authorization and Roles](authorization-and-roles.md)); a session that is older asks for the password or code again.

## Requests from other sites

A form on another site must not be able to make a signed-in user's browser run a Command. Punch refuses such a request with 403 `cross_origin_request` before it runs, using the headers browsers send, `Sec-Fetch-Site` and `Origin`, instead of form tokens. A request with an `Authorization` header is not checked: it carries no cookie the browser adds by itself. Other origins that may send Commands go in `PUNCH_TRUSTED_ORIGINS`.

## Changing the flows

The accounts slice is the app's code, not Punch's. Change a message, add a field to sign-up, send new users somewhere else after they sign in: edit the operation and its page like any other. Keep what the compiler and `punch check` keep for you: the rate limits on every Command that runs without a signed-in user, the `PasswordCheck` shape of a sign-in, and the same answer for every refused sign-in.

An app that started without a part can add it later: `punch generate accounts magic-links` writes it, and leaves a patch where files were changed ([Tooling](../tooling.md)).

## Testing

Specs sign in without a password, and hash with only 1,000 iterations, so passwords cost microseconds:

```crystal illustrative="the helpers; the accounts slice's specs use them throughout"
client = Punch::Spec.client.sign_in(user.id)          # a session, its cookie in the client
Punch::Spec.password_account(user.id, "ada@example.org", "correct horse battery")
token = Punch::Spec.api_token(user.id)
Punch::Spec.client.get(Accounts::Users::Me.path, headers: Punch::Spec.bearer(token))
```

The accounts slice's own spec of signing in:

```crystal source=docs/guides/examples/spec/accounts/sessions/create_spec.cr select="signing up and in with a browser signs in, and answers a wrong password like an unknown address"
# spec/accounts/sessions/create_spec.cr
# ...

describe "signing up and in with a browser" do
  it "signs in, and answers a wrong password like an unknown address" do
    sign_up(Punch::Spec.client)
    client = Punch::Spec.client
    client.visit(Accounts::Sessions::CreatePage.href).should have_element("input[name=password][autocomplete=current-password]")
    client.submit(Accounts::Sessions::Create.action, form: {"email" => "ingrid@example.org", "password" => PASSWORD})
      .should redirect_to(Home::IndexPage.href)
    client.visit(Home::IndexPage.href).status.should eq(HTTP::Status::OK)

    wrong = Punch::Spec.client.submit(Accounts::Sessions::Create.action, form: {"email" => "ingrid@example.org", "password" => "not my password"})
    unknown = Punch::Spec.client.submit(Accounts::Sessions::Create.action, form: {"email" => "nobody@example.org", "password" => PASSWORD})
    [wrong, unknown].each do |response|
      response.status.should eq(HTTP::Status::UNPROCESSABLE_ENTITY)
      response.should have_element(".form-errors li[data-code=invalid_credentials]", text: "The email address or password is wrong.")
      response.headers["Set-Cookie"]?.should be_nil
    end
  end

  # ...
end
```

## Common mistakes

**Reading the user from a cookie or header yourself.** Ask for the actor in `call`; Punch looks up the session in the operation's transaction.

**Hashing a password in `call`.** It holds the writer for 70 milliseconds per request. Take `Punch::Auth::NewPassword` in the input and store `input.password.hashed`.

**Saying "no such account".** A different answer for an unknown address tells an attacker which addresses have accounts. Return `Punch::Auth::InvalidCredentials` for every refusal, as the slice does.

**A sign-in without a rate limit per address.** It does not compile. Keep both limits the slice has, per address and per client address.

**Signing a token yourself.** A signed token cannot be revoked before it expires. Use `Punch::Auth::APITokens` or `Punch::Auth::OneTimeTokens`.

**Using a one-time token on the page behind the link.** A mail scanner's fetch would use it up. Look it up on the page; use it in the Command behind the button.

## Limits

- Sign-up says `taken` for an address that already has an account.
- API tokens have no scopes: a token can do what its user can.
- The second factor is an authenticator app with recovery codes; there are no passkeys and no SMS codes.

## Rules

- Ask for the actor in `call`; never read sessions or tokens yourself. ([The actor](#the-actor))
- Keep sessions, tokens, and links random and stored hashed, through Punch's APIs; never sign one. ([Sessions and API tokens](#sessions-and-api-tokens))
- Never hash or check a password inside a Command; take `NewPassword` and `PasswordCheck`. ([Checking a password](#checking-a-password))
- Give every refused sign-in the same answer, `Punch::Auth::InvalidCredentials`. ([Checking a password](#checking-a-password))
- Use a one-time token only in the Command behind the link's button. ([Links in mail](#links-in-mail))

## Reference

- [Authentication](../auth.md): tables, the actor, sessions, tokens, one-time tokens, identities, password fields, the second factor, events, the cross-site check, commands, and spec helpers.
- [Authorization](../authorization.md): policies and recent authentication.
- [Rate limits](../rate-limits.md): the limits on sign-in and sign-up.
