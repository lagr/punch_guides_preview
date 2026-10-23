# Translations and Formats

Applies to Punch 0.1.

A Punch app keeps every sentence for people in locale files, and code reaches them through generated methods: a misspelled key or a missing argument is a compile error, and `punch check` fails when a language lacks a text the default has. Programs never see those sentences. They read codes, `blank` or `already_sent`, and the app renders a message from the code at the edge, in the reader's language. Dates, numbers, and money are formatted per language from the same files.

After reading this guide, you will know:

- how to add text to a page without a string literal in the view;
- how to add a language, and what `punch check` asks of it;
- how Punch chooses the language of a request, and of a mail;
- how error codes become messages, in forms and in the API;
- how to format money, dates, and numbers for a reader;
- how to test text in several languages.

Before you start: you know [Pages and Components](pages-and-components.md) and [Results, Errors, and Status Codes](results-errors-and-status-codes.md). The examples come from the reference app, which speaks English and German.

## Locale files

Text lives in Rails-style YAML in `config/locales/`, one mapping per file under its locale tag. The reference app's invoice texts start like this:

```yaml source=reference/config/locales/en.yml select=en.invoices.numbered,en.invoices.draft,en.invoices.line_count
# config/locales/en.yml
en:
  # ...
  invoices:
    # ...
    numbered: "Invoice %{number}"
    draft: "Draft %{id}"
    # ...
    line_count:
      one: "%{count} line"
      other: "%{count} lines"
```

- **`%{name}`** is filled in from an argument of that name.
- **A plural** is a mapping whose keys are plural categories and that has `other`. English and German need `one` and `other`; Polish and Russian also `few` and `many`; Japanese only `other`. `zero` is allowed everywhere and wins for 0.
- **Any number of files** per locale, in subdirectories too. The accounts slice keeps its texts in `config/locales/accounts/`, so `punch upgrade:diff` can update them without touching yours.

Keys follow the app's structure: `invoices.*` for the invoice pages, `customers.*` for the customer pages, `layout.*` for the layout. A few namespaces Punch reads by name:

| Namespace | Holds |
|---|---|
| `errors.<code>` | The message of an error code in a rejection, filled in from its params (`%{max}`) |
| `problems.<code>` | The message of a whole problem, such as `already_sent` |
| `fields.<name>` | A form field's label |
| `llms` | The app's text in `/llms.txt` |
| `date`, `time`, `number` | Formats, as in Rails |
| `punch` | Punch's own messages and currency symbols, which an app may override |

## Generated accessors

The app declares its translations once, in `app/setup.cr`:

```crystal illustrative="one line of reference/app/setup.cr"
Punch.translations Invoicing::Translations, default: "en"
```

`punch i18n:generate` writes `app/translations.gen.cr`: one method per key, with the `%{…}` names as named arguments. `punch dev` and `punch typecheck` run it before each build, so a new key is there when you use it. Views reach the methods through `v.t`, other code through `locale.t`:

```crystal illustrative="calls from Billing::Invoices.title and LinesTable, reference/app/billing/invoices/components.cr"
t = locale.t.invoices
number ? t.numbered(number: number.to_s) : t.draft(id: id.to_s)

v.caption t.line_count(count: rows.size)
```

Because the accessors are methods, the compiler checks them:

```crystal fails=spec/fail/translation_unknown_key.cr
Punch.translations Fixture::Translations, default: "en"

Punch::I18n.default.t.inbox.unread
```

```text
Error: undefined method 'unread' for Fixture::Translations::Inbox
```

A missing argument fails the same way (`missing argument: name`). Arguments are strings and numbers; pass domain types with `.to_s`, and a plural's `count:` as an integer.

> [!NOTE]
> **Coming from Rails:** `t("invoices.numbered", number: n)` becomes `t.invoices.numbered(number: n)`. The key is a method, so a typo fails the build instead of rendering "translation missing" in production. See [Coming from Rails](coming-from-rails.md).

## Adding a language

A language is another set of locale files with the same keys. The `i18n` step of `punch check` compares them and fails on gaps:

```console
$ punch i18n:generate --check
punch: app/translations.gen.cr does not match the locale files. Run 'punch i18n:generate' and commit the result; never edit it by hand.
The translations have gaps:
  config/locales/en.yml:46:20: invoices.overdue_since is missing in de
```

It fails when:

- a locale lacks a key of the default locale, has one the default lacks, or uses other `%{…}` names for it;
- a plural lacks a form its locale's rule needs, or has one the rule never uses;
- a locale lacks Punch's own keys, which Punch ships in English and German only;
- the app's code uses an error code (`Punch::Invalid.new("…")`) or returns a failure result without a message in the default locale.

It also lists keys no code reaches, without failing, so dead text can be removed.

The languages an app offers come from its catalog, so a language picker needs no list of its own. The reference app's forms offer every locale, named in its own language:

```crystal source=reference/app/languages.cr select=Invoicing::Languages.options
# app/languages.cr
# The languages the app offers, for the forms that choose one, such as the
# interface's (`Accounts::Language::Update`).
module Invoicing::Languages
  # Each locale of the app, named in its own language.
  def self.options : Array({Punch::LocaleTag, String})
    Punch::I18n.catalog.locales.map do |code|
      {Punch::LocaleTag.parse(code).as(Punch::LocaleTag), Punch::I18n.locale(code).translate?("punch.language_name") || code}
    end
  end

  # ...
end
```

## Which language a request gets

A request's locale is, in order:

1. **The actor's stored choice.** The app's actor overrides `locale`, usually from a column of the user. The reference app stores it at sign-up and lets the user change it on the account page.
2. **`Accept-Language`**, highest `q` first. A tag matches exactly, else by language: `de-AT` finds `de`.
3. **The default locale.**

The response says which it chose:

```console
$ curl -i -H "Accept-Language: de-AT, en;q=0.5" http://localhost:3000/billing/invoices/1
HTTP/1.1 401 Unauthorized
Vary: Accept, Accept-Language
Content-Language: de

{"type":"about:blank","title":"Unauthorized","status":401,"code":"unauthenticated","request_id":"39fad561c750dbe9efced27c1ef3a7cc","detail":"Bitte melden Sie sich an."}
```

`Vary: Accept-Language` appears when the header decided, so caches keep the languages apart. An operation that needs the locale itself, such as sign-up storing it, asks for `locale : Punch::Locale` in `call`.

URL prefixes such as `/de/invoices` are not supported.

## Mail and jobs use the recipient's language

A mail is read by its recipient, not by whoever caused it. A job or subscriber therefore never uses the language of the request that triggered it. It builds the locale from what is stored about the recipient. The reference app mails an invoice in the customer's language, else the organization's:

```crystal illustrative="from Billing::Invoices::MailInvoice#call, reference/app/billing/invoices/subscribers/mail_invoice.cr"
locale = Punch::I18n.locale((customer.locale || organization.try(&.locale)).try(&.to_s))
```

`Punch::I18n.locale(tag)` gives the tag's locale, its language's, or the default. See [Mail](mail.md).

## Codes become messages at the edge

Errors carry codes and parameters, never sentences. A rejection says `out_of_range` with `{min: 1, max: 10000}`; a failure result is `AlreadySent`, code `already_sent`. The edge renders a message from the code:

- **In the API**, Problem Details gain `detail`, the problem's message, and each error a `message`. Programs keep reading `code`.
- **In forms**, each field shows its error's message, and a summary lists them all, with the code in `data-code`.
- **On problem pages**, the message is shown above the code.

The messages are the `errors.*` and `problems.*` keys:

```yaml source=reference/config/locales/en.yml select=en.problems
# config/locales/en.yml
en:
  # ...
  problems:
    already_sent: "This invoice was already sent."
    invalid_signature: "The webhook's signature does not match."
    no_lines: "A draft needs at least one line before it is sent."
    not_sent: "This invoice has not been sent yet."
    unknown_customer: "There is no such customer."
    unknown_invoice: "There is no such invoice."
    wrong_amount: "The amount does not match the invoice's total."
```

A code without a message renders as the code itself, and `punch check` fails for a code the app's own code uses without one. Punch's own codes (`blank`, `too_long`, `not_found`) come with messages in English and German, which an app may override.

## Formatting dates, numbers, and money

Every `Punch::Locale` formats values the way its files say:

| Call | English | German |
|---|---|---|
| `locale.date(time)` | `2026-03-05` | `05.03.2026` |
| `locale.date(time, :long)` | `March 5, 2026` | `5. März 2026` |
| `locale.time(time, :short)` | `05 Mar 14:07` | `05. März, 14:07 Uhr` |
| `locale.number(1234567)` | `1,234,567` | `1.234.567` |
| `locale.number(1234.5, 2)` | `1,234.50` | `1.234,50` |
| `locale.money(amount, eur)` | `€1,250.00` | `1.250,00 €` |

The invoice lines use all of them through `v.locale`:

```crystal illustrative="cells of Billing::Invoices::LinesTable"
v.td locale.number(line.quantity.value)
v.td locale.money(line.unit_price, currency)
v.td locale.money(line.total, currency)
```

- **Money** has as many decimals as its currency's minor unit: none for yen, two for euros, three for Kuwaiti dinars. The symbol comes from `punch.currencies.<CODE>` where the locale names one, else the code.
- **Times are not converted between zones.** Pass the time in the zone to show, with `time.in(location)` from the standard library.
- **Formats come from the locale files** (`date.formats`, `number.format`, `number.currency.format`), so files from rails-i18n fit. What the files do not say, Punch does not know: there is no CLDR.

Never format a value for people with `to_s`. `30000.to_s` is minor units, and `Time#to_s` is not a date anyone writes.

## Testing in several languages

`client.language = "de"` sends `Accept-Language` with every request of a spec client, and `Punch::I18n.locale("de")` is a locale for unit specs. The reference app checks its invoice titles and states in both languages:

```crystal source=reference/spec/billing/invoices/components_spec.cr select="Billing::Invoices names invoices and their states in each locale"
# spec/billing/invoices/components_spec.cr
describe Billing::Invoices do
  it "names invoices and their states in each locale" do
    en, de = Punch::I18n.locale("en"), Punch::I18n.locale("de")
    id = InvoicingSpec.parse(Billing::Invoices::ID, 7)
    number = InvoicingSpec.parse(Billing::Invoices::Number, "INV-000001")
    Billing::Invoices.title(en, nil, id).should eq("Draft 7")
    Billing::Invoices.title(de, number, id).should eq("Rechnung INV-000001")
    Billing::Invoices.state(de, :sent, overdue: true).should eq("überfällig")
    Billing::Invoices.state(en, :paid, overdue: false).should eq("paid")
  end
end
```

Field errors are matched by code, whatever the language: `have_field_error("name", "blank")` passes in English and in German.

## Common mistakes

**A sentence as a literal in a view.** `v.p "No invoices yet."` compiles, is English in every language, and no check finds it. Add the key to every locale file and write `v.p t.none`. This is one of the rules in the app's `AGENTS.md` that no tool checks yet.

**Building a sentence from parts.** `"#{t.invoice} #{number}"` assumes every language puts the number last. Make it one key with an argument: `numbered: "Invoice %{number}"`.

**A plural by `if count == 1`.** Other languages have more forms. Make the key a plural and pass `count:`.

**Sending a message to a client.** A problem type that holds a `message` string in English ties clients to one language and one wording. Return a code (the type's name) and put the message in `problems.<code>`.

**Formatting money with `to_s`.** Shows minor units. Use `v.locale.money(amount, currency)`.

**Using the request's language in a job.** A reminder enqueued while a German user worked goes to an English customer in German. Build the locale from the recipient's stored language.

**Editing `translations.gen.cr`.** The next `punch dev` overwrites it, and `punch check` fails before that. Change the locale files and run `punch i18n:generate`.

## Limits

- No URL prefixes for languages.
- No right-to-left layout support, and no translation management interface.
- No CLDR data: date and number formats are what the locale files say.
- Field names, operation IDs, and codes in action blocks for agents stay untranslated on purpose: agents copy them.
- The ops console is in English only.

## Rules

- Every sentence for people comes from the locale files, through `v.t` or `locale.t`. ([Generated accessors](#generated-accessors))
- Every locale has every key of the default locale; `punch check` enforces it. ([Adding a language](#adding-a-language))
- Errors and problems carry codes; their messages live in `errors.*` and `problems.*`. ([Codes become messages at the edge](#codes-become-messages-at-the-edge))
- Mail and jobs use the recipient's stored language, never the request's. ([Mail and jobs use the recipient's language](#mail-and-jobs-use-the-recipients-language))
- Format dates, numbers, and money through the locale, never with `to_s`. ([Formatting dates, numbers, and money](#formatting-dates-numbers-and-money))
- Never edit `translations.gen.cr`; change the locale files and run `punch i18n:generate`. ([Generated accessors](#generated-accessors))

## Reference

- [Translations and formats](../translations.md): locale files, accessors, locale choice, formats, and the checks in full.
- [Views and pages](../views.md#forms): labels and errors in forms.
- [JSON endpoints and OpenAPI](../openapi.md#problem-details): `detail` and `message` in Problem Details.
