# Input and Domain Types

Applies to Punch 0.1.

A Punch operation never sees raw request data. Before `call` runs, Punch parses the path, the query string, the form, or the JSON body into the operation's `Input`, a struct whose fields are already the types the domain uses: an `Email`, an invoice's `ID`, an amount of `Money`. If anything fails to parse, the operation does not run, and the client gets every error at once, each at its field. Inside `call`, there is nothing left to check that a type could have checked.

After reading this guide, you will know:

- how to declare an operation's input, and what a client sees when Punch refuses it;
- how to add rules that span several fields;
- how to write your own domain primitive, and keep its JSON Schema true;
- why an operation never holds a `String` that should have been an email address;
- how Punch represents money, and how to keep sums exact;
- how to test inputs and primitives.

Before you start: you know what a Query and a Command are ([Operations and routing](../operations.md)). The examples come from the reference app's billing slice, an invoicing app in `reference/`.

## Parse at the boundary, trust inside

Every value that crosses into an app is parsed once, where it arrives, into a type that can only hold valid values. Code past that point trusts the types and checks nothing again.

Here is the reference app's Command that adds a customer:

```crystal source=reference/app/billing/customers/operations/create.cr select=Billing::Customers::Create
# app/billing/customers/operations/create.cr
module Billing::Customers
  # ...

  # Adds a customer to the organization the actor works in. Each
  # organization's customers have unique email addresses.
  @[Punch::Authorize(Accounts::Organizations::Current)]
  struct Create < Punch::Command
    struct Input
      include Punch::Input

      getter name : Punch::NonEmptyString
      getter company_name : Punch::NonEmptyString?
      getter email : Punch::Email
      # The language of the customer's mail; without one, the user's.
      getter locale : Punch::LocaleTag?
    end

    alias Output = Created | EmailTaken | Punch::Rejection

    def call(input : Input, db : Punch::Writer, actor : Invoicing::Actor, _grant : Punch::Grant(Create), clock : Punch::Clock) : Output
      if rejection = Invoicing::Languages.check(input.locale)
        return rejection
      end

      case created = Table.insert(db, organization_id: actor.organization_id, name: input.name, company_name: input.company_name, email: input.email,
        created_at: clock.now, locale: input.locale)
      in Record     then Created.new(Customer.new(created))
      in EmailTaken then created # the unique index decides; 422 at `email`
      end
    end
  end
end
```

The input declares four fields. `name` is a `Punch::NonEmptyString`, so a name of only spaces never reaches `call`. `email` is a `Punch::Email`, already trimmed and lowercased. `company_name` and `locale` are nilable, so a request may leave them out. The body of `call` neither validates the fields nor normalizes them. Its one check is one no type can make, because it depends on the app: whether the app offers the language asked for. Then it inserts what it was given.

> [!NOTE]
> **Coming from Rails:** there are no model validations. What `validates :email, format: …` does in Active Record is the job of the field's type here, checked before the operation runs. Rules that need the database, such as "this email is not taken yet", are decided by the database itself and come back as a typed result ([Queries](../queries.md#violations)).

## Declaring an input

An input is a struct named `Input` inside the operation that includes `Punch::Input`. Each `getter` is a field. Three rules decide where a field comes from and whether it is required:

1. A field marked `@[Punch::PathParam]` comes from the URL path, and only from there.
2. The other fields come from the JSON body for a Command, from the query string for a Query or a Page, and from the form body for a Command that a page's form sends.
3. A field is required unless its type is nilable (`String?`) or it has a default (`= 1`).

The reference app's Command that starts an invoice takes a customer, a currency, and lines:

```crystal source=reference/app/billing/invoices/operations/create.cr select=Billing::Invoices::Create::Input
# app/billing/invoices/operations/create.cr
module Billing::Invoices
  # ...

  # Starts a draft for a customer of the organization the actor works in.
  # Another organization's customer is unknown, like one that does not
  # exist, even where the actor is a member: drafts go into the current
  # organization only.
  @[Punch::Authorize(Accounts::Organizations::Current)]
  struct Create < Punch::Command
    struct Input
      include Punch::Input

      getter customer_id : Customers::ID
      getter currency : Punch::Currency
      getter lines : Array(LineInput) = [] of LineInput

      def check(errors : Punch::Rejection::Collector) : Nil
        Invoices.check_lines(lines, errors)
      end
    end

    # ...
  end
end
```

`lines` defaults to an empty array, so a draft may start without lines. Each line is itself an input, declared once and reused by the Commands that create and revise drafts:

```crystal source=reference/app/billing/invoices/inputs.cr
# app/billing/invoices/inputs.cr
module Billing::Invoices
  # One line of an invoice in a request.
  struct LineInput
    include Punch::Input

    getter description : Punch::NonEmptyString
    getter quantity : Quantity
    # Price of one unit in minor units of the invoice's currency.
    getter unit_price : Punch::Money

    def check(errors : Punch::Rejection::Collector) : Nil
      max = Invoice::MAX_UNIT_PRICE
      errors.add("unit_price", Punch::Invalid.new("out_of_range", min: -max, max: max)) if unit_price.value.abs > max
    end

    def to_line : Invoice::Line
      Invoice::Line.new(description, quantity, unit_price)
    end
  end

  # Rejects more lines than an invoice may have.
  def self.check_lines(lines : Array(LineInput)?, errors : Punch::Rejection::Collector) : Nil
    if lines && lines.size > Invoice::MAX_LINES
      errors.add("lines", Punch::Invalid.new("too_many", max: Invoice::MAX_LINES))
    end
  end
end
```

A field can be a `String`, `Bool`, `Int32`, `Int64`, `Float64`, `Time` (RFC 3339 with an offset), an enum (its member's snake_case name), a domain primitive, a nested input, or an array of any of these. A `Bool` field is declared with `getter?`, so the method reads `input.urgent?`. Any other type does not compile, and the message names the field:

```crystal fails=spec/fail/input_union_type.cr
struct Amount
  include Punch::Input
  getter value : Int32 | String
end

Amount.from_json("{}")
```

```text
Error: Punch: Amount#value has type (Int32 | String), which Punch::Input cannot parse.
Use String, Bool, Int32, Int64, Float64, Time, an enum, a domain primitive (Punch::Primitive),
a nested Punch::Input, or an Array of one of these. Make it nilable (`T?`) to make it optional.
```

How each type is written in JSON, in a query string, and in a form is in [Input: field types](../input.md#field-types).

## What a refused input looks like

When an input does not parse, Punch answers `422 Unprocessable Entity` with a Problem Details body whose `code` is `invalid_input`. Its `errors` hold every error found, not just the first. Each error has a `path` to the field, a `code`, the code's `params`, and a `message` in the request's language.

The reference app's spec for creating customers checks exactly that:

```crystal source=reference/spec/billing/customers/create_spec.cr select="customers API collects every input error"
# spec/billing/customers/create_spec.cr
# ...

describe "customers API" do
  # ...

  it "collects every input error" do
    response = AccountsSpec.client.post(Billing::Customers::Create.path, json: {name: " ", email: "not an address"})
    response.status.should eq(HTTP::Status::UNPROCESSABLE_ENTITY)
    response.json["errors"].as_a.map { |error| {error["path"][0].as_s, error["code"].as_s} }.should eq([{"name", "blank"}, {"email", "invalid_email"}])
  end
end

# ...
```

Paths reach into nested inputs and arrays: an error in the second line's quantity is at `["lines", 1, "quantity"]`. An error whose path is empty belongs to the input as a whole.

Codes are the contract; messages are for people. A client branches on `code`, never on `message`, which changes with the language and the wording. The messages come from the app's locale files, so an app can reword any of them ([Translations](../translations.md)).

## What Punch ignores, and what it refuses

Parsing is strict where leniency would hide a bug, and lenient where clients need room:

- **Unknown fields are ignored**, in JSON bodies and query strings alike. Clients may send more than an input declares, and links may carry tracking parameters. The price: a misspelled optional field is dropped without an error.
- **A repeated key is refused** with `duplicate_field`, because parsers disagree about which value wins. In a query string, repeating a key is how an array is sent (`ids=4&ids=5`).
- **`null` in JSON** counts as absent for an optional field, and as `required` for any other.
- **An empty value in a query string or a form** counts as absent, because that is what HTML forms send. An unchecked checkbox sends nothing at all, so a checkbox needs a `Bool` with a default: `getter? urgent : Bool = false`.
- **Form rows that are entirely empty are left out**, since forms show empty rows to fill in. An error still names the row as the form numbered it.

The full list is in [Input: strictness](../input.md#strictness).

## Rules across fields

Some rules involve more than one field: an end date after a start date, at most a hundred lines. Override `check` in the input. It runs only after every field has parsed, so it can use the fields' types without checking for `nil`:

```crystal illustrative="a rule between two fields; the reference app's inputs have no date ranges"
def check(errors : Punch::Rejection::Collector) : Nil
  errors.add("ends_on", Punch::Invalid.new("ends_before_start")) if ends_on < starts_on
end
```

`errors.add` takes the path the error belongs to and a `Punch::Invalid`: a snake_case code and its parameters, never a sentence. In the reference app, `LineInput#check` above bounds the unit price, and `Invoices.check_lines` bounds the number of lines. Both errors land in the same rejection as the parse errors, and the client sees them all together.

A `check` decides what one request may hold. A rule about the stored state, such as "only a draft may be sent", is not an input rule: it belongs to the model, which the operation asks after loading the row ([Domain models](../models.md)).

## Domain primitives

A domain primitive is a type for one kind of value that can only be created by parsing. Once code holds a `Punch::Email`, it holds a valid email address: there is no other way to get one.

That is why an operation never takes a `String` for an email address and checks it in `call`. A `String` field says nothing about what it holds. Every function it reaches has to check it again or trust that someone did, and the check that one function forgets is the bug. A primitive moves the check into the type, where the compiler makes everyone go through it.

Punch provides these primitives:

| Type | Holds | Some of its codes |
|---|---|---|
| `Punch::NonEmptyString` | text with a visible character, trimmed | `blank`, `invalid_characters` |
| `Punch::Email` | an address, international forms included, lowercased | `invalid_email`, `too_long` |
| `Punch::Slug` | `annual-report-2026`: lowercase ASCII letters, digits, single hyphens | `invalid_slug`, `too_long` |
| `Punch::Money` | an amount in the currency's minor unit | `out_of_range` |
| `Punch::Currency` | an ISO 4217 code such as `EUR` | `invalid_currency` |
| `Punch::LocaleTag` | a language tag such as `de` or `pt-BR` | `invalid_locale` |
| `Punch::PageSize` | how many rows a list returns per page | `out_of_range` |

Every text primitive refuses invalid UTF-8 and control characters, strips surrounding whitespace, and normalizes to Unicode NFC. So "Café" typed with a single "é" or with "e" and a combining accent is one value, and a unique index treats it as one ([Text](../text.md#normalization)).

### Writing your own

Include `Punch::Primitive(T)`, where `T` is the type the value is stored and sent as, and define `self.parse`. It returns the primitive or a `Punch::Invalid`. The reference app has two, an invoice number and a line's quantity:

```crystal source=reference/app/billing/invoices/types.cr select=Billing::Invoices::Number,Billing::Invoices::Quantity
# app/billing/invoices/types.cr
module Billing::Invoices
  # ...

  # The number a sent invoice carries, `INV-000042`: assigned in sending
  # order without gaps, at least six digits.
  struct Number
    include Punch::Primitive(String)

    FORMAT = /\AINV-[0-9]{6,}\z/

    def self.json_schema
      {pattern: "^INV-[0-9]{6,}$"}
    end

    def self.parse(raw : String) : self | Punch::Invalid
      FORMAT.matches?(raw) ? new(raw) : Punch::Invalid.new("invalid_invoice_number")
    end

    # The number after *sequence*, the numeric part of the last one issued.
    def self.next(sequence : Int64) : self
      new("INV-#{(sequence + 1).to_s.rjust(6, '0')}")
    end
  end

  # How many units a line bills, 1 to 10,000.
  struct Quantity
    include Punch::Primitive(Int64)

    MAX = 10_000_i64

    def self.json_schema
      {minimum: 1, maximum: MAX}
    end

    def self.parse(raw : Int) : self | Punch::Invalid
      raw >= 1 && raw <= MAX ? new(raw.to_i64) : Punch::Invalid.new("out_of_range", min: 1, max: MAX)
    end
  end

  # ...
end
```

`parse` calls `new`, which is private. The compiler enforces both halves: a primitive without `self.parse` does not compile, and neither does one with a public `initialize`:

```crystal fails=spec/fail/primitive_public_initialize.cr
struct PublicNew
  include Punch::Primitive(String)

  def initialize(@value : String)
  end

  def self.parse(raw : String) : self | Punch::Invalid
    new(raw)
  end
end
```

```text
Error: PublicNew#initialize must be private: a domain primitive is created only by `self.parse` (lib/punch/docs/core-types.md).
```

A primitive can still have other constructors of its own that call `new`, as `Number.next` does: those are part of the type, and the type decides what is valid.

Primitives compare and hash by value, print and serialize as their value, and can be fields of inputs, columns of tables ([Queries](../queries.md#domain-types-columnscr)), and payloads of jobs.

### Its JSON Schema

A primitive's `self.json_schema` returns JSON Schema keywords, such as `pattern` or `minimum`. They describe the field in the OpenAPI document and set the attributes of form fields, so a browser can refuse a bad value before sending it.

Nothing makes `json_schema` and `parse` agree by itself. A spec does, with `Punch::Spec.schema_agrees`: it parses each sample and checks the schema accepts exactly the valid ones. The reference app's:

```crystal source=reference/spec/billing/invoices/types_spec.cr select="Billing primitives"
# spec/billing/invoices/types_spec.cr
# ...

describe "Billing primitives" do
  it "describes invoice numbers as it parses them" do
    Punch::Spec.schema_agrees(Billing::Invoices::Number,
      valid: ["INV-000001", "INV-999999", "INV-1000000"],
      invalid: ["", "INV-", "INV-00001", "inv-000001", "INV-00000a", " INV-000001", "INV-000001 ", "XINV-000001"])
  end

  it "describes quantities as it parses them" do
    Punch::Spec.schema_agrees(Billing::Invoices::Quantity,
      valid: [1_i64, 2_i64, 9_999_i64, Billing::Invoices::Quantity::MAX],
      invalid: [0_i64, -1_i64, Billing::Invoices::Quantity::MAX + 1, Int64::MAX, Int64::MIN])
  end
end
```

Give a spec like it to every primitive that declares `json_schema`, and include the edge cases: the limits, one past each limit, and strings with surrounding spaces.

## Typed IDs

Every kind of record has its own ID type, a subclass of `Punch::TypedID`:

```crystal source=reference/app/billing/invoices/types.cr select=Billing::Invoices::ID
# app/billing/invoices/types.cr
module Billing::Invoices
  struct ID < Punch::TypedID
  end

  # ...
end
```

An invoice's ID cannot be passed where a customer's ID is expected: the compiler refuses it. A typed ID is a positive `Int64`, the SQLite rowid, and it parses only from its canonical decimal form, with no sign, leading zeros, or spaces. As a path parameter, `@[Punch::PathParam] getter invoice_id : ID` gives `call` an ID that is at least well-formed. Whether the invoice exists is still for the operation to find out.

## Money

`Punch::Money` is an integer number of the currency's minor unit: cents for euros, yen for yen. Sums never round. On the wire it is a JSON integer, so `1250` in euros is 12.50 EUR, and in the database it is an `INTEGER` column.

The currency travels once per document, as a `Punch::Currency` field beside the amounts, not with every amount. The invoice input above shows the pattern: `currency` on the invoice, `unit_price` on each line. Punch does not add amounts of two currencies for you; keep one currency per document.

An amount stays within ±(2^53 − 1), the integers a JavaScript client reads exactly. `+`, `-`, and `*` raise `OverflowError` beyond that, which is a bug, not a result. So bound whatever feeds a sum. The reference app bounds lines per invoice, quantity per line, and the unit price, which keeps every possible total far inside the range:

```crystal source=reference/app/billing/invoices/invoice.cr select=Billing::Invoices::Invoice::MAX_LINES,Billing::Invoices::Invoice::MAX_UNIT_PRICE
# app/billing/invoices/invoice.cr
# Tier 2 (lib/punch/docs/models.md): an invoice has a lifecycle, and its state decides what
# may happen to it. Each state is its own immutable type; a transition is a
# method that returns the next state. Invoices::Store turns rows into these
# types and back. Nothing here touches the database.
module Billing::Invoices::Invoice
  # Lines per invoice, and the largest unit price in minor units: with
  # Quantity::MAX they keep every total far inside Punch::Money's range.
  MAX_LINES      =               100
  MAX_UNIT_PRICE = 1_000_000_000_i64

  # ...
end
```

`Quantity` enforces its maximum in `parse`, and `LineInput#check` and `Invoices.check_lines` enforce the other two, as shown earlier.

## Secrets

Values that must not leak, such as API keys, tokens, and passwords, are wrapped in `Punch::Secret(T)`. A secret prints as `[REDACTED]` in logs, error reports, and interpolated strings. Serializing one does not compile:

```crystal fails=spec/fail/secret_to_json.cr
{token: Punch::Secret.new("sk_live_123")}.to_json
```

```text
Error: Punch::Secret cannot be serialized to JSON. Call `unwrap` explicitly if the value really must leave the process (lib/punch/docs/core-types.md).
```

Call `unwrap` where the value is used, such as the line that sets an HTTP header, and nowhere earlier. The credentials an app declares hand out their secrets this way ([Credentials](../credentials.md)). The wrapper prevents accidents, not attacks on the process: it does not wipe the value from memory.

## Testing inputs and primitives

Test an input through its endpoint when the point is what a client sees, as the customer spec above does. To check one primitive or one rejection directly, `require "punch/spec"` adds two matchers:

```crystal illustrative="the matchers on their own; the reference app's specs above use them in context"
Billing::Invoices::Number.parse("INV-42").should be_invalid("invalid_invoice_number")
Billing::Invoices::Quantity.parse(0_i64).should be_invalid("out_of_range")
rejection.should be_rejected("lines", 1, "quantity", code: "out_of_range")
```

An input has no keyword constructor: it exists only by parsing. When a spec needs one, for example to call an operation directly, `Punch::Spec.input` parses plain values and fails the example with every error if they do not parse:

```crystal illustrative="a spec that calls an operation directly; the reference app's specs go through endpoints"
input = Punch::Spec.input(Billing::Invoices::Send::Input, {"invoice_id" => "42"})
```

The first argument after the type holds the path parameters as strings; the other fields follow as named arguments.

## Common mistakes

**Taking a `String` and checking it in `call`.** The input declares `getter email : String`, and `call` checks the format or lowercases it. The check runs in one operation and is missing in the next. Declare the field as `Punch::Email`, or as a primitive of your own, and delete the check.

**A field of a type Punch cannot parse.** A union such as `Int32 | String`, a `Hash`, or a flags enum fails to compile with the message shown in [Declaring an input](#declaring-an-input). Pick one type; for a value of several shapes, see [Limits](#limits).

**Expecting an error for a misspelled field.** A client sends `{"compnay_name": "…"}`, and the customer is created without a company. Unknown fields are ignored on purpose. Test the fields a client relies on, and make them required where the operation cannot do without them.

**A checkbox bound to a required `Bool`.** An unchecked box sends nothing, so the field comes back `required`. Give it a default: `getter? urgent : Bool = false`.

**Raising for invalid input.** Code in `call` raises when a value is wrong, and the client gets a 500. An expected failure is a result. Put the rule in the field's type or in `check`, or return a problem type from `call` ([Operations and routing](../operations.md)).

**Floats for amounts.** `getter price : Float64` rounds sums. Use `Punch::Money` in minor units, with the currency once per document.

**Making a primitive's `initialize` public "for the tests".** The compile error above refuses it, and a test that builds an invalid value proves nothing. Build test values with `parse`.

## Limits

- An input of one of several shapes, such as a payment by card or by bank transfer, each with its own fields, is not supported yet. Use optional fields and a `check` that requires the right ones for the chosen kind.
- `Hash` fields, unions other than `T?`, and flags enums are not input types.
- `check` does not run for an input that code builds with its own `initialize`, only for one that is parsed.
- `Punch::Money` keeps no currency and does not convert between currencies.
- `Punch::Secret` prevents accidental leaks only. It does not wipe the value from memory, and an unwrapped value is an ordinary value.

## Rules

- Give every input field the narrowest type that fits: a domain primitive, a typed ID, `Punch::Money`, an enum. A `String` is for free text only. ([Domain primitives](#domain-primitives))
- Never re-check in `call` what a field's type guarantees. ([Parse at the boundary, trust inside](#parse-at-the-boundary-trust-inside))
- Put rules across fields in the input's `check`; put rules about stored state in the model. ([Rules across fields](#rules-across-fields))
- Errors are codes with parameters, never sentences. Clients branch on `code`. ([What a refused input looks like](#what-a-refused-input-looks-like))
- A primitive's `initialize` is private; create one only through `parse` or the type's own constructors. ([Writing your own](#writing-your-own))
- Every primitive that declares `json_schema` has a `Punch::Spec.schema_agrees` spec. ([Its JSON Schema](#its-json-schema))
- Amounts are `Punch::Money`; bound every quantity and price that feeds a sum. ([Money](#money))
- Wrap anything that must not leak in `Punch::Secret`, and `unwrap` it only where it is used. ([Secrets](#secrets))

## Reference

- [Input](../input.md): every field type in each encoding, strictness, and every error code.
- [Core types](../core-types.md): results, `Invalid` and `Rejection`, primitives, typed IDs, money, and secrets.
- [JSON Schema](../json-schema.md#domain-primitives): the keywords primitives declare.
- [Text](../text.md#normalization): normalization and collations.
- [Testing](../testing.md): the spec helpers.
