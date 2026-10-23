# Credentials and Keys

Applies to Punch 0.1.

An app needs things from its deployment: a payment provider's API key, the company name on its invoices, the mail server, and one root secret to sign links with. In Punch, the app declares all of them as one type. Each value is looked up in the environment first, then in an encrypted file committed beside the code, and every missing or malformed value stops the boot with a list of all of them. Nothing reads `ENV` directly, and nothing secret ends up in a log.

After reading this guide, you will know:

- how to add an API key or a setting without reading `ENV`;
- where values come from, and what is committed;
- how to set, show, and check credentials;
- how to sign a link that needs no database row;
- how to rotate the root key without breaking a single link.

Before you start: you know what a `Punch::Secret` is ([Input and Domain Types](input-and-domain-types.md#secrets)).

## Declaring what the app needs

`app/credentials.cr` declares one type, `punch new` writes it, and `app/setup.cr` names it with `Punch.credentials`. The reference app's:

```crystal source=reference/app/credentials.cr select=Invoicing::Credentials
# app/credentials.cr
# What the app needs from its deployment, checked at boot (lib/punch/docs/credentials.md): each
# value from the environment first (ISSUER__NAME), then from the encrypted
# config/credentials/<env>.yml.enc. `bin/invoicing creds:check` shows where
# each came from.
struct Invoicing::Credentials
  include Punch::Credentials

  # Signs invoice links (`Invoices::Sharing`); rotated with `punch keys:rotate`.
  getter secret_key_base : Punch::KeyRing
  getter issuer : Issuer
  # The payment provider's webhooks (`Billing::Payments::Provider`).
  getter payments : Payments
  # How mail leaves the app (`Billing::Invoices::MailInvoice`): captured
  # in development (MAIL__DELIVERY=capture), sent by SMTP in production.
  getter mail : Punch::Mailer::Settings

  # Who issues the invoices; shown on each one.
  struct Issuer
    include Punch::Credentials

    getter name : Punch::NonEmptyString
    getter vat_id : Punch::NonEmptyString?
  end

  struct Payments
    include Punch::Credentials

    # The provider's signing secret (`whsec_…`), and the one before it
    # while the provider rotates them. Without one the app takes no
    # payments.
    getter webhook_secret : Punch::Secret(String)?
    getter previous_webhook_secret : Punch::Secret(String)?

    # Every secret a delivery may be signed with; `nil` without any.
    def webhook_secrets : Array(String)?
      secret = webhook_secret || return
      [secret.unwrap, previous_webhook_secret.try(&.unwrap)].compact
    end
  end
end
```

- **Each field is a value the deployment provides.** A group is a nested type that includes `Punch::Credentials`, such as `issuer`.
- **A field is required unless it is nilable or has a default.** `payments.webhook_secret` is nilable: without it, the app takes no payments.
- **Anything that must not leak is a `Punch::Secret(T)`.** It prints as `[REDACTED]`, cannot be serialized, and is unwrapped where it is used.
- **Field types** are the ones inputs use: strings, numbers, booleans, enums, domain primitives, secrets of those, and groups. Anything else does not compile.

Code reads them anywhere after boot: `Invoicing::Credentials.current.issuer.name`. `current` is set once, before the server starts. The app's mail settings are a group too, `Punch::Mailer::Settings` ([Mail](mail.md#settings)).

Reading `ENV` instead is reported by `punch check` (`Punch/EnvAccess`):

```text
Declare this setting in the app's credentials type (Punch::Credentials) instead of reading ENV (lib/punch/docs/credentials.md)
```

A setting read from `ENV` is checked nowhere: a typo in its name, or a missing value, shows up when that line first runs in production. A declared credential is checked at every boot, with all its siblings.

## Where values come from

Every value is looked up in two places, in this order:

1. **The environment.** The variable is the field's path in capitals, joined by `__`: `issuer.name` is `ISSUER__NAME`, `payments.webhook_secret` is `PAYMENTS__WEBHOOK_SECRET`.
2. **The environment's encrypted file,** `config/credentials/<env>.yml.enc`, compiled into the binary.

| File | Holds | Committed |
|---|---|---|
| `config/credentials/<env>.yml.enc` | the YAML, encrypted with AES-256-GCM | yes |
| `config/credentials/<env>.key` | the key that opens it | never |

`PUNCH_ENV` selects the environment, `development` by default. In a deployment, the key comes from `PUNCH_MASTER_KEY` instead of a file. Because the encrypted file is compiled in, changing a value in it takes a build; a value that must change without a deploy belongs in the environment, which is looked up first.

A key in the file that no field declares stops the boot too, so a misspelt key cannot silently leave a default in place.

## Setting and checking values

The `punch creds:*` commands edit an environment's file through its key:

```console
$ punch creds:init --env production
$ echo "Example GmbH" | punch creds:set issuer.name
$ pass payments/webhook | punch creds:set payments.webhook_secret --env production
$ punch creds:show --env production
$ punch creds:edit
```

`creds:set` reads the value from standard input, so it stays out of the shell's history. `creds:show` redacts every value unless asked with `--reveal`. `creds:edit` opens the YAML in your editor and refuses to write invalid YAML.

`bin/<app> creds:check` loads the credentials as `serve` would and shows where each value came from, the environment or the file, without showing secrets. `punch check` makes sure no key file is committed, every file whose key is present opens, and the built app's `creds:check` passes ([Credentials: commands](../credentials.md#commands)).

In specs, `Punch::Spec.credentials(Type, yaml)` in `spec/spec_helper.cr` loads the type from a YAML string. Specs need no key and no file.

## The key ring

One credential is special: `secret_key_base : Punch::KeyRing`, the app's root secret. Punch derives a separate key from it for each purpose, so a key that signs invoice links signs nothing else, and a signing key never encrypts.

The ring signs the few values that must be stateless, that is, valid without a row in the database. The reference app's shared invoice links are one:

```crystal source=reference/app/billing/invoices/sharing.cr
# app/billing/invoices/sharing.cr
module Billing::Invoices
  # Links that show a sent invoice without signing in: the invoice ID,
  # signed with the app's key ring for this purpose only, valid for
  # `VALID`. Nothing is stored; a key rotation keeps old links working
  # until the old key is retired (lib/punch/docs/keys.md).
  module Sharing
    PURPOSE = "invoice_link"
    VALID   = 30.days

    def self.token(invoice : Invoice::Issued, expires_at : Time) : String
      Punch::KeyRing.current.signer(PURPOSE).sign_text(invoice.id.value.to_s, expires_at: expires_at)
    end

    # The invoice ID in *token*, or `nil` when it is forged, damaged, or
    # expired.
    def self.invoice_id(token : String, now : Time) : ID?
      Punch::KeyRing.current.signer(PURPOSE).verify_text(token, now).try { |raw| ID.parse(raw).as?(ID) }
    end
  end
end
```

- `signer("invoice_link")` gives the key for that purpose alone; a token signed for `invoice_link` never verifies as anything else.
- The expiry is part of what is signed.
- Verification returns `nil` for anything that is forged, damaged, expired, or signed for another purpose, without saying which.
- **Signed is not hidden.** Anyone with the link can read the invoice ID in it. Sign IDs, never secrets.

The ring also encrypts, with `encryptor(purpose)`. Punch uses it for the second factor's secrets and for the cookie that shows a new API token once.

What the ring is not for: sessions, API tokens, and one-time links. Those are random values stored hashed ([Authentication](authentication.md#sessions-and-api-tokens)), so rotating the ring signs nobody out and invalidates no password reset.

## Rotating keys without breaking links

A rotation takes three steps, each followed by a build and a deploy:

| Step | Command | Effect |
|---|---|---|
| 1 | `punch keys:rotate` | adds a new key, pending: every process accepts it, none signs with it |
| 2 | `punch keys:promote` | the new key signs; the old one is still accepted |
| 3 | `punch keys:retire ID` | removes the old key: what it signed no longer verifies |

Step 1 exists for the deploy: while the old and the new process both run, the old one must already accept what the new one signs. Retire the old key once nothing it signed is still needed. For invoice links valid for 30 days, that is 30 days after step 2. The ops console's keys page shows how often each old key is still used ([Key ring: observing](../keys.md#observing)).

After a compromise, `punch keys:rotate --revoke` replaces every key at once. Everything signed or encrypted under the old keys stops opening, so it is a last resort.

## Common mistakes

**Reading `ENV["STRIPE_KEY"]`.** It is checked nowhere and reported by `punch check`. Declare a field.

**Committing a key file.** Anyone with the repository could open the credentials. `punch check` refuses a committed or unignored key file; `punch creds:init` adds the ignore rule.

**A secret as a plain `String`.** It can end up in a log, an error, or a JSON response. Declare it as `Punch::Secret(String)`.

**Putting a secret in a signed token.** Signing does not hide. Sign an ID, and look the rest up.

**Retiring a key right after promoting.** Every link it signed stops working. Wait out the longest validity.

**Using the key ring for sessions or reset links.** Use the random, hashed tokens Punch has for them.

## Limits

- Credentials are read once at boot; there is no reloading.
- `Punch::Secret` keeps values out of logs and responses, not out of memory.
- Changing a value in the encrypted file takes a build and a deploy.

## Rules

- Declare every setting and secret the app needs in its `Punch::Credentials` type; never read `ENV`. ([Declaring what the app needs](#declaring-what-the-app-needs))
- Make anything that must not leak a `Punch::Secret`. ([Declaring what the app needs](#declaring-what-the-app-needs))
- Never commit a key file; set values with `punch creds:set`, from standard input. ([Setting and checking values](#setting-and-checking-values))
- Sign only what must be stateless, one purpose per signer, and sign IDs, never secrets. ([The key ring](#the-key-ring))
- Rotate in three deploys, and retire a key only after its longest-lived token has expired. ([Rotating keys without breaking links](#rotating-keys-without-breaking-links))

## Reference

- [Credentials](../credentials.md): field types, parsing, the file format, commands, and specs.
- [Key ring](../keys.md): purposes, formats, rotation, observing, and limits.
