# Mail

Applies to Punch 0.1.

An app mails people: an invoice to a customer, a link to reset a password, a notice that an address was added. In Punch, a mail is a component with recipients and a subject. One view yields both the HTML and the plain-text part, in the recipient's language, and a job delivers it, so a slow or failing mail server never holds up a request and a failed delivery is retried.

After reading this guide, you will know:

- how to write a mail, with its subject, recipients, and both parts;
- how to send it, and why a Command never sends it itself;
- how to configure delivery for development and production;
- how to see mail in development without sending it;
- how to test a mail and follow its links.

Before you start: you know what a job and a deferred subscriber are ([Background Jobs](background-jobs.md), [Domain Events](domain-events.md)).

## A mail is a component

A mail is a struct that includes `Punch::Mail`. It says who it goes to, what its subject is, and what it shows. The reference app's invoice mail:

```crystal source=reference/app/billing/invoices/mail/invoice_mail.cr
# app/billing/invoices/mail/invoice_mail.cr
module Billing::Invoices
  # The mail that brings a sent invoice to its customer: the amount, the
  # due date, and a link that shows the invoice without signing in.
  struct InvoiceMail
    include Punch::Mail

    def initialize(@invoice : Invoice::Issued, @customer : Customers::Record, @link : Punch::Href, @issuer : String)
    end

    def to : Punch::Mail::Address
      Punch::Mail::Address.new(@customer.email, @customer.display_name)
    end

    def subject(locale : Punch::Locale) : String
      locale.t.invoices.mail.subject(number: @invoice.number.to_s, issuer: @issuer)
    end

    def view(v : Punch::View) : Nil
      t = v.t.invoices.mail
      number = @invoice.number.to_s
      v.p t.greeting(name: @customer.display_name)
      v.p t.body(number: number, total: v.locale.money(@invoice.total, @invoice.currency), due: v.locale.date(@invoice.due_at, :long))
      v.p { v.a t.open(number: number), href: @link }
      v.p t.validity
      v.p do
        v.text t.closing
        v.br
        v.text @issuer
      end
    end
  end
end
```

- `to` returns a `Punch::Mail::Address` or an array of them. An address takes a `Punch::Email`, so it is valid by construction. `reply_to` is optional. The sender is the same for every mail and comes from the settings.
- `subject` takes the locale the mail is rendered in. Text comes from the locale files, as everywhere in an app.
- `view` writes the body with the same view as a page's components ([Views and pages](../views.md#components)). Dates and money go through `v.locale`, in the recipient's formats.

One view gives both parts. The HTML backend writes plain semantic HTML, without styles, which mail clients show in their own way. The text backend lays the same elements out as plain text: paragraphs separated by blank lines, links as `text <https://…>`, lists with dashes. There is no template to keep in sync with another.

A link is `SomePage.href(…)`, as in a page. In a mail, it becomes an absolute URL below `PUNCH_PUBLIC_URL`, where people reach the app.

A mail cannot hold a form, and `v.raw` does not exist in a mail's view: a mail links to the page that has the form.

## Sending from a job

`Punch::Mailer.deliver(mail, locale:, job:)` renders the mail and delivers it. It takes the job's context, so mail is only sent from jobs, and inside a write transaction it raises. Normally the job is a deferred subscriber of the event that calls for the mail. The reference app mails an invoice when it is sent:

```crystal source=reference/app/billing/invoices/subscribers/mail_invoice.cr
# app/billing/invoices/subscribers/mail_invoice.cr
module Billing::Invoices
  # Mails a sent invoice to its customer. Deferred (lib/punch/docs/events.md): a job
  # through the outbox, so the mail goes out exactly when the invoice was
  # sent, after the response, and is retried if delivery fails.
  #
  # The mail is in the customer's language, else in the organization's,
  # else in the app's default.
  @[Punch::Subscriber::Delivery(:deferred)]
  @[Punch::Job::Options(queue: "mail")]
  struct MailInvoice
    include Punch::Subscriber(InvoiceSent)

    def call(event : InvoiceSent, job : Punch::Job::Context) : Punch::Job::Result
      invoice, customer, organization = job.read do |db|
        customer = Customers::Table.find(db, event.customer_id)
        {Store.load(db, event.invoice_id), customer, customer.try { |found| Accounts::Organizations::Table.find(db, found.organization_id) }}
      end
      return discard("invoice #{event.invoice_id} is gone") unless invoice.is_a?(Invoice::Issued) && customer

      expires_at = job.now.at_beginning_of_second + Sharing::VALID
      link = SharedPage.href(token: Sharing.token(invoice, expires_at))
      locale = Punch::I18n.locale((customer.locale || organization.try(&.locale)).try(&.to_s))
      mail = InvoiceMail.new(invoice, customer, link, Invoicing::Credentials.current.issuer.name.value)
      Punch::Mailer.deliver(mail, locale: locale, job: job)
    end
  end
end
```

`deliver` returns what `perform` returns: `nil` when delivered, a retry when delivery failed and may succeed later (no connection, a temporary refusal), and a discard when the server refused the recipient for good. So the subscriber returns it as it is.

Why never from a Command? Talking to a mail server takes time and can fail. Inside a Command, it would hold the database's write lock while it waits, and a mail sent for a Command that then rolls back cannot be recalled. Through the outbox, the mail exists exactly when the Command's writes do, and is retried until it goes out.

**The recipient's language, not the request's.** A mail is read by its recipient, often long after the request that caused it. Store a language with whoever receives mail, and render in it. The reference app uses the customer's language, else the organization's, else the app's default.

Jobs run at least once, so after a crash between delivering and recording the job as done, a mail can go out twice. SMTP has no idempotency key; keep the window small by doing the slow work before `deliver`.

## Settings

The app declares its mail settings as a group in its credentials:

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

| Credential | Environment variable | |
|---|---|---|
| `mail.delivery` | `MAIL__DELIVERY` | `capture`, `smtp`, or `adapter`; required |
| `mail.from` | `MAIL__FROM` | the sender's address; required |
| `mail.from_name` | `MAIL__FROM_NAME` | the sender's name |
| `mail.smtp.host`, `.port` | `MAIL__SMTP__HOST`, `MAIL__SMTP__PORT` | the server, port 587 by default |
| `mail.smtp.username`, `.password` | `MAIL__SMTP__USERNAME`, `MAIL__SMTP__PASSWORD` | sent only after TLS |
| `mail.smtp.tls` | `MAIL__SMTP__TLS` | `starttls` (the default; it must succeed), `smtps` for TLS from the start, or `plain` for a relay on the same host |

The server's certificate and name are verified. A setting that does not fit, such as `smtp` without a host, stops the boot with the other credential problems ([Credentials](../credentials.md)).

For a provider's HTTP API instead of SMTP, set `delivery: adapter` and register one of Carbon's provider adapters once, in `app/setup.cr` ([Mail: settings](../mail.md#settings)).

`PUNCH_PUBLIC_URL`, such as `https://invoices.example.org`, is the base of every link in mail. Development and test default it; every other environment requires it.

## Seeing mail in development

With `delivery: capture`, which `punch new` sets for development and test, nothing is sent. The process keeps the newest 500 mails in memory, and the ops console shows them at `/_punch/mail`: sender, recipients, the text part as a mail client shows it, and a link to the HTML part. Sign up in the browser, and the verification mail is there.

## Testing mail

Specs capture every mail, whatever the settings say, so they never send. The `deliver_mail` matcher checks what a block delivered, and `Punch::Spec.mails` gives the mails themselves:

```crystal source=reference/spec/billing/invoices/mail_invoice_spec.cr select="sending an invoice mails it to the customer through the outbox, with a link that needs no sign-in"
# spec/billing/invoices/mail_invoice_spec.cr
# ...

describe "sending an invoice" do
  it "mails it to the customer through the outbox, with a link that needs no sign-in" do
    client = AccountsSpec.client
    email = unique_email
    id, number = sent_invoice(client, email)
    Punch::Spec.mails.should be_empty # not before the job runs

    -> { Punch::Spec.perform_jobs.map(&.type).should eq(["Billing::Invoices::MailInvoice(Billing::InvoiceSent)"]) }
      .should deliver_mail(Billing::Invoices::InvoiceMail, to: email, times: 1)
    mail = Punch::Spec.mails(to: email).first
    mail.subject.should eq("Invoice #{number} from Example Consulting GmbH")
    mail.from.should eq(%("Example Consulting" <invoices@example.org>))
    mail.to.should eq([%("Ada Lovelace" <#{email}>)])
    mail.locale.should eq("en")
    mail.text.should contain("Dear Ada Lovelace,\n\nplease find invoice #{number} of €250.00, due on ")
    (mail.html.try(&.value) || "").should contain("<p>Dear Ada Lovelace,</p>")
    link = mail.paths.first? || fail "no link in #{mail.text}"
    mail.text.should contain("View invoice #{number} <#{Punch.public_url}#{link}>")
    shared = Punch::Spec.client.get(link)
    shared.status.should eq(HTTP::Status::OK)
    shared.json["number"].should eq(number)
    shared.json["id"].should eq(id.value)
    Punch::Spec.client.visit(link).should have_element("h1", text: /#{number}/)
  end

  # ...
end
```

The spec runs the job, checks the subject, sender, recipient, language, and both parts, then follows the mail's link with the spec client, as the customer would. A captured mail's `paths` are its links below `PUNCH_PUBLIC_URL`, ready for the client. `Punch::Spec.render_mail(mail, locale)` renders a mail without delivering it, for a spec of the mail alone.

## Common mistakes

**Sending from a Command.** `Punch::Mailer.deliver` takes a job's context, so it does not fit in `call`. Publish an event and mail from a deferred subscriber, or enqueue a job.

**Rendering in the request's language.** The person who clicked is often not the one who reads the mail. Store each recipient's language and render in it.

**Building links from strings.** A path written by hand breaks when the page moves, and a relative link does not work in a mail client. Use `Page.href(…)`, which Punch makes absolute.

**Using Carbon directly.** `Carbon::Email`, its development and SMTP adapters, and `deliver_later` bypass the job and its retries; `punch check` reports them (`Punch/MailDelivery`). Write a `Punch::Mail` and send it through `Punch::Mailer`.

**Inline styles.** The view refuses `style`. Mail clients show plain HTML in their own way, and so does the text part.

## Limits

- Attachments are not supported.
- SMTP has no idempotency key: after a crash at the wrong moment, a mail can go out twice.
- Captured mail lives in the process's memory: a restart clears it, and each process has its own.

## Rules

- Write mail as a `Punch::Mail` component, with text from the locale files. ([A mail is a component](#a-mail-is-a-component))
- Send only from a job, normally a deferred subscriber, with `Punch::Mailer.deliver`, and return its result. ([Sending from a job](#sending-from-a-job))
- Render every mail in its recipient's language. ([Sending from a job](#sending-from-a-job))
- Link with `Page.href(…)`; set `PUNCH_PUBLIC_URL` in every deployed environment. ([Settings](#settings))
- Never use Carbon's email class or its development and SMTP adapters directly. ([Common mistakes](#common-mistakes))

## Reference

- [Mail](../mail.md): the text layout of each element, settings, adapters, tracing, and spec helpers.
- [Domain events](../events.md): deferred subscribers.
- [Credentials](../credentials.md): declaring and setting the mail settings.
