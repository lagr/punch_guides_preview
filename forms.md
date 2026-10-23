# Forms

Applies to Punch 0.1.

A form in Punch is bound to a Command. Its action is the Command's typed path, its fields are the Command's input fields, checked by the compiler, and what the browser sees after sending it follows one fixed set of conventions: a redirect when the Command succeeds, the same page again with every error at its field when it does not. You never write a field name, a URL, or an error message by hand, and the form, the JSON API, and the agent's action block all run the same Command.

After reading this guide, you will know:

- how to add a form for a Command, and the page that holds it;
- what the browser gets after sending a form, for success and for each kind of refusal;
- where a form's errors come from, and how they are shown;
- how to fill a form with values, repeat rows, and choose controls;
- how to make a form safe to send twice;
- how to test forms.

Before you start: you know [Pages and Components](pages-and-components.md) and [Input and Domain Types](input-and-domain-types.md). The examples come from the reference app's customer and invoice pages.

## A form for a Command

The reference app's page for adding a customer:

```crystal source=reference/app/billing/customers/pages/create_page.cr
# app/billing/customers/pages/create_page.cr
module Billing::Customers
  # The fields a customer is made of, for adding and for editing.
  struct NewView
    include Punch::Component

    def initialize(@actor : Invoicing::Actor)
    end

    def view(v : Punch::View) : Nil
      t = v.t.customers
      v.render Invoicing::Layout.new(t.new_customer, @actor) do
        v.h1 t.new_customer
        v.form(Create.action, idempotent: true) do |form|
          form.field :name, label: t.contact_person
          form.field :company_name, hint: t.company_hint
          form.field :email
          form.field :locale, as: :select, options: Invoicing::Languages.options, label: t.mail_language, hint: t.mail_language_hint
          form.submit t.add
        end
      end
    end
  end

  # The form that adds a customer: `GET /billing/customers/new`.
  @[Punch::Authorize(Accounts::Organizations::Current)]
  struct CreatePage < Punch::Page
    struct Input
      include Punch::Input
    end

    alias Output = NewView

    def call(input : Input, db : Punch::Reader, actor : Invoicing::Actor) : Output
      NewView.new(actor)
    end

    def self.redirect(result : Created) : Punch::Href
      IndexPage.href
    end
  end
end
```

- **`v.form(Create.action, …)`** binds the form to `Billing::Customers::Create`. The action is the Command's path, checked like `Create.path`; for a Command with path parameters, pass them: `Send.action(invoice_id: id)`.
- **`form.field :name`** writes a labelled control for the input field `name`: a `<div class="field">` with the label, the control, the hint, and the field's errors. The symbol is checked against `Create::Input` at compile time, so `form.field :emial` does not compile.
- **The label** is `fields.<name>` from the locale files, or made from the name (`company_name` is "Company name"); `label:` replaces it. **`hint:`** adds a line under the control, linked by `aria-describedby`.
- **`form.submit`** writes the button.

The control follows the field's type:

| Field | Control |
|---|---|
| `Bool` with a default | a checkbox; unchecked sends nothing |
| other `Bool`, enums | a `select`; an optional one starts with an empty choice |
| An array of an enum | `select multiple` |
| Integers, floats, and primitives over them (IDs, `Money`) | `number`, with `min` and `max` from the primitive's schema |
| `String`, `Time`, other primitives | `text`, with `minlength` and `maxlength` from the schema; email fields get the email keyboard |

`as:` chooses another control: `:textarea`, `:password` (never filled in, also not after a refusal), `:hidden`, or `:select` with `options:` as value and label pairs, as the language field above does. A required field (neither nilable nor defaulted) is marked `required`, so browsers check it before sending; the server checks again.

## The page that holds a form

Every form has a page that holds it, so that a refusal has somewhere to be shown. By default it is the page named after the Command: `CreatePage` holds `Create`'s form, `SendPage` would hold `Send`'s. A page that shows forms for other Commands lists them:

```crystal illustrative="the annotations of Billing::Invoices::ShowPage, which shows Send and Destroy buttons"
@[Punch::Forms(Send, Destroy)]
struct ShowPage < Punch::Page
```

A form without a holding page does not compile:

```crystal fails=spec/fail/form_without_page.cr
module Notes
  @[Punch::Public]
  @[Punch::RateLimitExempt("a fixture")]
  struct Create < Punch::Command
    struct Input
      include Punch::Input
      getter title : String
    end

    def call(input : Input, db : Punch::Writer) : Nil
    end
  end
end

struct NoteForm
  include Punch::Component

  def view(v : Punch::View) : Nil
    v.form(Notes::Create.action) { |form| form.field :title }
  end
end

Punch::HTMLView.render(NoteForm.new)
```

```text
Error: Punch::View#form: no page holds the form of Notes::Create
```

The holding page is shown again from the Command's path alone, so its path parameters must be the Command's and its other input fields optional; that is checked at compile time too. `punch routes` shows which page holds each Command's form:

```text
POST  /billing/invoices/:invoice_id/send  json  Billing::Invoices::Send  (form on Billing::Invoices::ShowPage)
```

A Command whose form a page holds accepts form bodies (`application/x-www-form-urlencoded`) besides JSON. One without a holding page answers a form body with 415: it is an API endpoint only.

## After sending: 303 or the page again

What the browser gets depends on what the Command returned:

| The Command | The browser gets |
|---|---|
| succeeds | `303 See Other` to the page's `redirect(result)`, or to the holding page itself |
| refuses the input, or returns a rejection or a unique violation | `422` and the holding page again, with what was sent and each field's errors |
| returns another failure (`409 AlreadySent`, `422 NoLines`) | that status and the holding page again, the failure's message as the form's error |
| needs a signed-in user | 303 to the sign-in page |

These are Turbo Drive's conventions: Turbo follows a 303 and renders a 4xx answer to a form in place. Browsers without Turbo do the same with a full page load. Nothing is different for JSON clients: the same Command answers JSON as before.

Where to go on success is the holding page's `redirect`, overloaded per result type. Adding a customer leads back to the list; starting an invoice leads to the new invoice:

```crystal source=reference/app/billing/invoices/pages/create_page.cr select=Billing::Invoices::CreatePage
# app/billing/invoices/pages/create_page.cr
module Billing::Invoices
  # ...

  # `GET /billing/invoices/new`, optionally for one customer.
  @[Punch::Authorize(Accounts::Organizations::Current)]
  struct CreatePage < Punch::Page
    struct Input
      include Punch::Input

      getter customer_id : Customers::ID?
    end

    alias Output = NewView

    def call(input : Input, db : Punch::Reader, actor : Invoicing::Actor) : Output
      customers = Customers::Queries.page(db, organization_id: actor.organization_id, order: :name, limit: 100).map { |record| Customers::Customer.new(record) }
      NewView.new(customers, input.customer_id, actor)
    end

    def self.redirect(result : Created) : Punch::Href
      ShowPage.href(invoice_id: result.invoice.id)
    end
  end
end
```

A `redirect(result : Nil)` handles a Command that returns `nil`, such as deleting a draft. Without a matching `redirect`, the browser goes back to the holding page at the Command's path parameters.

> [!NOTE]
> **Coming from Rails:** this is the `if @customer.save then redirect_to … else render :new, status: :unprocessable_entity` of a Rails controller, written once by Punch for every form. There is no controller action to write. See [Coming from Rails](coming-from-rails.md).

## Field errors

Here is what the browser gets back when a customer is added with a name of only a space:

```html illustrative="the <main> of the 422 answer from the reference app, line breaks added"
<form id="billing-customers-post" method="post" action="/billing/customers">
<div class="form-errors" role="alert"><p>Please correct the following:</p>
<ul>
<li data-code="blank"><a href="#billing-customers-post-name">Name</a>: Must not be empty.</li>
</ul>
</div>
<input type="hidden" name="_idempotency_key" value="DVPyNocp683u_ub3dmUgvnuo">
<div class="field field-invalid">
<label for="billing-customers-post-name">Contact person</label>
<input type="text" name="name" id="billing-customers-post-name" required aria-invalid="true" aria-describedby="billing-customers-post-name-error" value=" ">
<div class="field-errors" id="billing-customers-post-name-error">
<p class="field-error" data-code="blank">Must not be empty.</p>
</div>
</div>
<div class="field">
<label for="billing-customers-post-email">Email</label>
<input type="text" name="email" id="billing-customers-post-email" required value="ada@example.test" maxlength="254" inputmode="email" autocomplete="email"></div>
...
<button type="submit">Add customer</button></form>
```

- **What was sent is shown again**, so nothing typed is lost (except passwords).
- **Each field has its errors**, linked with `aria-invalid` and `aria-describedby` for screen readers, and a summary at the top links to each field.
- **Messages come from the locale files**, in the reader's language, from the error's code (`blank`). The code stays in `data-code`, for specs and for scripts.

The errors come from three places, all handled the same way:

- **The input.** A field whose type refuses the value: `Punch::NonEmptyString` refuses a blank name with `blank`.
- **The input's `check`.** Rules across fields, such as at most 100 lines.
- **The database.** A unique index the insert hit becomes the error `taken` at its field. Adding a second customer with the same email shows "is already taken" under the email field, without the Command having looked anything up.

Only the form that was sent shows the values and errors. A page with a delete button per row marks the right row, since forms are told apart by Command and path.

## Filling a form

`values:` fills a form, as a named tuple checked at compile time against the input: an unknown field, a path parameter, or a value of the wrong type does not compile. The invoice form starts with the customer the page was opened for and euros:

```crystal illustrative="from Billing::Invoices::NewView, quoted below"
v.form(Create.action, values: {customer_id: @customer_id, currency: currency}, idempotent: true) do |form|
```

An edit page fills the form with the record's current values the same way, and its Command takes nilable fields, so a field left as it was is sent again unchanged.

## Rows and custom layouts

`form.field` fits most fields. For other layouts, `form.label`, `form.control`, and `form.errors` write the parts alone. `form.each` writes rows of a nested array, such as an invoice's lines in a table:

```crystal source=reference/app/billing/invoices/pages/create_page.cr select=Billing::Invoices::NewView
# app/billing/invoices/pages/create_page.cr
module Billing::Invoices
  # The form that starts a draft: a customer, a currency, and lines, with
  # empty rows to fill in. Empty rows are left out when it is sent.
  struct NewView
    include Punch::Component

    BLANK_LINES = 3

    def initialize(@customers : Array(Customers::Customer), @customer_id : Customers::ID?, @actor : Invoicing::Actor)
    end

    def view(v : Punch::View) : Nil
      t = v.t.invoices
      v.render Invoicing::Layout.new(t.new_invoice, @actor) do
        v.h1 t.new_invoice
        if @customers.empty?
          v.p do
            v.text "#{t.add_customer_first} "
            v.a t.new_customer, href: Customers::CreatePage.href
          end
          next
        end
        currency = Punch::Currency.parse("EUR").as(Punch::Currency)
        v.form(Create.action, values: {customer_id: @customer_id, currency: currency}, idempotent: true) do |form|
          form.field :customer_id, as: :select, options: @customers.map { |customer| {customer.id, customer.display_name} }
          form.field :currency, hint: t.currency_hint
          v.table(class: "lines") do
            v.thead do
              v.tr do
                v.th t.description, scope: "col"
                v.th t.quantity, scope: "col"
                v.th t.unit_price_cents, scope: "col"
              end
            end
            v.tbody do
              form.each(:lines, LineInput, blank: BLANK_LINES) do |line, index|
                v.tr do
                  v.td do
                    line.control :description, aria: {label: t.line_description(line: index + 1)}
                    line.errors :description
                  end
                  v.td do
                    line.control :quantity, aria: {label: t.line_quantity(line: index + 1)}
                    line.errors :quantity
                  end
                  v.td do
                    line.control :unit_price, aria: {label: t.line_unit_price(line: index + 1)}
                    line.errors :unit_price
                  end
                end
              end
            end
          end
          form.submit t.create_draft
        end
      end
    end
  end

  # ...
end
```

`form.each(:lines, LineInput, blank: 3)` writes the rows the values hold, then three empty ones. Empty rows are not sent as rows. A refused line marks its own field, `lines[0][quantity]`, with its error. Give controls in a table an `aria` label that says which row they belong to.

`form.fields(:address, Address) { |address| … }` writes one nested input the same way.

## Buttons for state changes

A change of state that needs no fields is a form with only a button. The invoice page sends and deletes drafts that way:

```crystal illustrative="from Billing::Invoices::DetailView"
v.form(Send.action(invoice_id: @detail.id)) { |form| form.submit t.send }
v.form(Destroy.action(invoice_id: @detail.id), class: "danger", data: {turbo_confirm: t.delete_confirm}) { |form| form.submit t.delete }
```

`data: {turbo_confirm: …}` makes Turbo ask before sending. A `DELETE` or `PATCH` Command gets `?_method=DELETE` in its action, because a form can only `POST`; the router honours it for form bodies only.

**Forms ask the policy.** On a page that takes the actor, a form for a Command the viewer may not run is left out, so a member who may read invoices but not send them sees no Send button. There is no `if can?(…)` in the component. See [Authorization and Roles](authorization-and-roles.md).

## Forms that are safe to send twice

A double click, or a retry after a lost answer, sends a form twice. `idempotent: true` makes that harmless: the form carries a generated idempotency key in a hidden field (`_idempotency_key`), and the second request gets the first one's answer instead of running the Command again. Both forms that add things in the reference app use it, and a form for a Command that requires a key always gets one.

A refused form is shown again with the same key and is not stored, since it changed nothing, so the corrected form still goes through. See [Building a JSON API](building-a-json-api.md#safe-retries-with-idempotency-keys).

## Cross-site requests

Every Command is protected against forms on other sites without tokens in your forms. A request from a browser page on another site is refused with 403 `cross_origin_request` before the Command runs, by the browser's `Sec-Fetch-Site` and `Origin` headers. There is no CSRF token to add or forget. See [Authentication](authentication.md#requests-from-other-sites).

## Testing forms

A spec submits a form as a browser does and checks the answer:

```crystal source=reference/spec/billing/customers/create_page_spec.cr select="customer forms shows the form again with what was sent and why it was refused"
# spec/billing/customers/create_page_spec.cr
# ...

describe "customer forms" do
  # ...

  it "shows the form again with what was sent and why it was refused" do
    client = AccountsSpec.client
    customer(client)
    refused = client.submit(Billing::Customers::Create.action, form: {"name" => " ", "email" => "Ada@Example.org"})
    refused.status.should eq(HTTP::Status::UNPROCESSABLE_ENTITY)
    refused.should have_field_error("name", "blank")
    refused.should have_element("input[name=email][value='Ada@Example.org']")

    taken = client.submit(Billing::Customers::Create.action, form: {"name" => "Ada", "email" => "Ada@Example.org"})
    taken.status.should eq(HTTP::Status::UNPROCESSABLE_ENTITY)
    taken.should have_field_error("email", "taken")
    taken.should have_element("input[name=name][value=Ada]")
  end
end
```

- `client.submit(Command.action, form: {...})` sends a form body as a browser does; `values:` instead of `form:` checks the values at compile time.
- `have_field_error("email", "taken")` finds the error with that code at that field; for a row, `have_field_error("lines[0][quantity]", "out_of_range")`.
- `response.should redirect_to(IndexPage.href)` checks the 303.

The same spec file checks the success path and the double submit:

```crystal source=reference/spec/billing/customers/create_page_spec.cr select="customer forms adds one customer when the form is sent twice, by its idempotency key"
# spec/billing/customers/create_page_spec.cr
# ...

describe "customer forms" do
  # ...

  it "adds one customer when the form is sent twice, by its idempotency key" do
    client = AccountsSpec.client
    form = client.visit(Billing::Customers::CreatePage.href)
    key = form.html.find("form[action='/billing/customers'] input[type=hidden][name=_idempotency_key]").first["value"]
    fields = {"name" => "Grace Hopper", "email" => "grace@example.org", Punch::IdempotencyKeys::FIELD => key}
    first = client.submit(Billing::Customers::Create.action, form: fields)
    first.should redirect_to(Billing::Customers::IndexPage.href)
    client.submit(Billing::Customers::Create.action, form: fields).should be_replay_of(first)
    client.get(Billing::Customers::Index.path).json["customers"].as_a.size.should eq(1)
  end

  # ...
end
```

## Common mistakes

**Writing field names or URLs by hand.** `<input name="emial">` in raw markup is not possible, and `form.field :emial` does not compile:

```text
Error: no overload matches 'Punch::View::Form(Notes::Create::Input)#field' with type Symbol
```

Use the input's field names; the compiler lists them.

**A form without a holding page.** `v.form(Create.action)` on a page that is not `CreatePage` and does not list `Create` in `@[Punch::Forms(...)]` does not compile. Add the annotation to the page that shows the form.

**Checking in the page before the Command runs.** Hiding the Send button for a draft without lines does not stop a second tab from sending one. Let the Command refuse, and the form shows its `NoLines` message.

**Looking up uniqueness in the page or the Command.** A unique index decides; its violation becomes the field error `taken` by itself.

**A checkbox bound to a required `Bool`.** An unchecked box sends nothing, so the field comes back `required`. Give it a default: `getter? urgent : Bool = false`.

**Expecting a 200 after success.** A successful form answers 303. A spec that expects 200 tests the wrong thing; check `redirect_to`.

**A link to a Command.** Commands are reached by forms only. A link would be a `GET`, which a crawler or a prefetch can follow.

## Limits

- No file uploads. Multipart bodies are refused.
- An input of one of several shapes (a payment by card or by transfer, each with its own fields) is not supported yet; use optional fields and a `check`.
- `Time` fields are text in RFC 3339; locale formats are for showing times, not for reading them.

## Rules

- Bind every form to a Command with `v.form(Command.action)`; never write field names or URLs by hand. ([A form for a Command](#a-form-for-a-command))
- Every form has a holding page: the page named after the Command, or one that lists it in `@[Punch::Forms]`. ([The page that holds a form](#the-page-that-holds-a-form))
- Choose the page after success with `redirect`, per result type. ([After sending: 303 or the page again](#after-sending-303-or-the-page-again))
- Leave validation to the input's types, `check`, and the database; the form shows their errors. ([Field errors](#field-errors))
- Make forms that create things or move money `idempotent: true`. ([Forms that are safe to send twice](#forms-that-are-safe-to-send-twice))
- Let forms ask the policy; never hide a button with your own permission check. ([Buttons for state changes](#buttons-for-state-changes))

## Reference

- [Views and pages](../views.md#forms): the form builder, controls, holding pages, and what follows a submit.
- [Input](../input.md): form bodies and field types.
- [Translations](../translations.md): labels and error messages.
- [Testing](../testing.md): `client.submit`, `redirect_to`, and `have_field_error`.
