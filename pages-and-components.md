# Pages and Components

Applies to Punch 0.1.

People use a Punch app through pages. A page is an operation like a Query: it takes an input and a database reader, calls the app's operations, and answers with HTML. The HTML is written by components, plain Crystal structs that write elements one by one through a view. There is no template language. A component's arguments are type-checked, a renamed field is a compile error instead of a blank spot on a page, and every value is escaped unless the code says otherwise in a way a reviewer can see.

After reading this guide, you will know:

- how to write a page that calls operations and renders components;
- how to write components, and share them between pages;
- how to build a layout with navigation;
- how to link between pages without writing a URL;
- why there is no template language, and what that buys you;
- how to test pages and components.

Before you start: you know [Operations and Routing](operations-and-routing.md) and [Results, Errors, and Status Codes](results-errors-and-status-codes.md). The examples come from the reference app's invoice pages, in `reference/app/billing/invoices/pages/`.

## A page calls operations and renders components

Here is the page that shows one invoice:

```crystal source=reference/app/billing/invoices/pages/show_page.cr select=Billing::Invoices::DetailView,Billing::Invoices::ShowPage elide=live
# app/billing/invoices/pages/show_page.cr
module Billing::Invoices
  @[Punch::Status(200)]
  struct DetailView
    include Punch::Component

    def initialize(@detail : Detail, @customer : String?, @actor : Invoicing::Actor)
    end

    def view(v : Punch::View) : Nil
      t = v.t.invoices
      title = Invoices.title(v.locale, @detail.number, @detail.id)
      v.render Invoicing::Layout.new(title, @actor, morph: true) do
        v.h1 title
        v.dl(class: "facts") do
          v.dt t.customer
          v.dd @customer || t.unknown_customer
          v.dt t.status
          v.dd Invoices.state(v.locale, @detail.status, @detail.overdue?)
          v.render Dates.new(@detail)
        end
        v.p t.draft_note if @detail.status.draft?
        v.render LinesTable.new(@detail)
        if @detail.status.draft?
          v.div(class: "actions") do
            v.form(Send.action(invoice_id: @detail.id)) { |form| form.submit t.send }
            v.form(Destroy.action(invoice_id: @detail.id), class: "danger", data: {turbo_confirm: t.delete_confirm}) { |form| form.submit t.delete }
          end
        end
      end
    end
  end

  # One invoice, with its customer's name from a second query. A draft
  # can be sent or deleted from here; a refusal (a draft without lines)
  # shows this page again with the reason. Agents read it too, as
  # markdown with the two forms as action blocks. Open in a browser, it
  # shows a change made elsewhere (sent, paid, revised, deleted) by
  # refreshing itself.
  @[Punch::AgentReadable]
  @[Punch::Authorize(ByRole)]
  @[Punch::Forms(Send, Destroy)]
  struct ShowPage < Punch::Page
    alias Input = Show::Input

    alias Output = DetailView | NotFound

    def call(input : Input, db : Punch::Reader, actor : Invoicing::Actor, clock : Punch::Clock) : Output
      case detail = Show.new.call(input, db, actor, clock)
      in NotFound then detail
      in Detail
        customer = Customers::Table.find(db, detail.customer_id).try(&.display_name)
        DetailView.new(detail, customer, actor)
      end
    end

    def live(event : InvoiceChanged, input : Input, db : Punch::Reader, actor : Invoicing::Actor, clock : Punch::Clock) : Punch::TurboStream?
      # ...
    end

    # After a deleted draft, the list; after sending, this page again.
    def self.redirect(result : Nil) : Punch::Href
      IndexPage.href
    end
  end
end
```

A page is the third kind of operation, beside Query and Command:

| | Query | Command | Page |
|---|---|---|---|
| Receives | `Punch::Reader` | `Punch::Writer` | `Punch::Reader` |
| Answers | `GET`, JSON | `POST`, `PATCH`, `DELETE`, JSON | `GET`, HTML |
| In `openapi.json` | yes | yes | no |

`ShowPage` shares its input with the Query `Show` (`alias Input = Show::Input`), calls it, and renders the result. The rules stay in the operations and the model; the page decides only what to show. Keep pages that thin. When a page needs something no operation provides, write a Query for it, so the JSON API gets it too.

`call` returns components or failures. `DetailView` is a component with status 200; `NotFound` is a failure, rendered as Punch's problem page with status 404 and the problem's message in the reader's language. Returning data that is neither does not compile:

```crystal fails=spec/fail/page_result_data.cr
struct Note
  include JSON::Serializable
  getter title = "x"

  def initialize
  end
end

@[Punch::Public]
struct NotePage < Punch::Page
  struct Input
    include Punch::Input
  end

  def call(input : Input, db : Punch::Reader) : Note
    Note.new
  end
end

Punch::Routing.routes.first.handler
```

```text
Error: Note is a page's result, but not a component and not a failure
```

Data for programs belongs to a Query.

## Where pages live, and their routes

A page's name ends in `Page`, and each page has its own file in its namespace's `pages/` directory: `Billing::Invoices::ShowPage` in `app/billing/invoices/pages/show_page.cr`. Without the suffix, its name gives its route by the same rules as any operation, always with `GET`:

| Page | Route |
|---|---|
| `Billing::Invoices::IndexPage` | `GET /billing/invoices` |
| `Billing::Invoices::ShowPage` | `GET /billing/invoices/:invoice_id` |
| `Billing::Invoices::CreatePage` | `GET /billing/invoices/new` |
| `Billing::Invoices::UpdatePage` | `GET /billing/invoices/:invoice_id/edit` |

`ShowPage` and the Query `Show` share the path `/billing/invoices/:invoice_id`. The router picks by `Accept`: a browser, which prefers `text/html`, gets the page; `curl` and `fetch`, which send `*/*` or nothing, get the JSON. Both answers carry `Vary: Accept`. `punch routes` shows each route's surfaces:

```text
GET   /billing/invoices/:invoice_id  json  Billing::Invoices::Show
GET   /billing/invoices/:invoice_id  html  Billing::Invoices::ShowPage
```

A page named like a Query, without the suffix, does not compile:

```text
Error: Notes::Show is a Page, so its name ends in Page, such as ShowPage.
```

## Components

A component includes `Punch::Component`, takes what it shows in its constructor, and writes it in `view(v : Punch::View)`. Components several pages render live in the namespace's `components.cr`; the invoice lines are shown on the invoice's page and on the page a customer opens from a shared link:

```crystal source=reference/app/billing/invoices/components.cr select=Billing::Invoices::LinesTable
# app/billing/invoices/components.cr
module Billing::Invoices
  # ...

  # The lines of an invoice with their total: shown to the user who issued
  # it and to the recipient of a shared link.
  struct LinesTable
    include Punch::Component

    def initialize(@detail : Detail)
    end

    def view(v : Punch::View) : Nil
      t, locale = v.t.invoices, v.locale
      currency, rows = @detail.currency, @detail.lines
      v.table(class: "lines") do
        v.caption t.line_count(count: rows.size)
        v.thead do
          v.tr { [t.position, t.description, t.quantity, t.unit_price, t.amount].each { |label| v.th label, scope: "col" } }
        end
        v.tbody do
          rows.each do |line|
            v.tr do
              v.td line.position.to_s
              v.td line.description.to_s
              v.td locale.number(line.quantity.value)
              v.td locale.money(line.unit_price, currency)
              v.td locale.money(line.total, currency)
            end
          end
        end
        v.tfoot do
          v.tr do
            v.th t.total, scope: "row", colspan: 4
            v.td { v.strong locale.money(@detail.total, currency) }
          end
        end
      end
    end
  end

  # ...
end
```

- **Elements are methods named like HTML's:** `v.table`, `v.thead`, `v.tr`, `v.th`, `v.td`, `v.strong`. Each takes its text or a block, and attributes as named arguments (`class: "lines"`, `scope: "col"`, `colspan: 4`).
- **Text comes from the locale files**, through `v.t`, and amounts and numbers are formatted by `v.locale`. A component never holds a sentence as a string literal ([Translations and Formats](translations-and-formats.md)).
- **One component renders another** with `v.render(LinesTable.new(detail))`.

A component that one page uses lives in that page's file. `DetailView` is in `show_page.cr`, above `ShowPage`, because nothing else renders it.

## Why no template language

Templates are strings with holes. The compiler cannot see into them: a misspelled method, a field renamed in a refactor, a value of the wrong type all surface when someone opens the page. Components are code, so all of those are compile errors.

They also make the safe thing the only easy thing:

- **Text and attribute values are always escaped.** `v.td line.description.to_s` writes `Consulting &lt;on site&gt;`, never markup. There is no `html_safe` to forget and no `raw` call in a template to miss in review.
- **Inline scripts and styles do not compile.** An event handler or a `style:` attribute fails at compile time, and the Content-Security-Policy would block them in the browser anyway:

  ```crystal fails=spec/fail/view_event_handler.cr
struct Bad
  include Punch::Component

  def view(v : Punch::View) : Nil
    v.div(onclick: "alert(1)") { }
  end
end

Punch::HTMLView.render(Bad.new)
  ```

  ```text
  Error: Punch::View: no `onclick`. Inline event handlers are blocked by the Content-Security-Policy
  ```

  Behaviour goes in a JavaScript module with a Stimulus controller or a plain `data-` attribute; looks go in the stylesheet with a class ([Assets and JavaScript](assets-and-javascript.md)).
- **URLs are typed.** An `href`, `src`, or `action` takes a `Punch::Href`, never a string ([Links between pages](#links-between-pages)).
- **Nesting follows HTML.** A list holds only `li`, a table only its sections and rows, a paragraph only text and text elements, a link no other link. A mistake raises `Punch::View::NestingError` when the component renders, so any spec that renders it finds the mistake. The nesting rules are narrowed to what markdown can express too, so the same components can serve agents ([Pages for Agents](pages-for-agents.md)).

> [!NOTE]
> **Coming from Rails:** where ERB has a view, partials, and helpers, Punch has components for all three. A partial is a component a page renders; a helper is a module function such as `Billing::Invoices.state`. See [Coming from Rails](coming-from-rails.md).

When markup really must be written as it is, such as an inline SVG icon, a `chrome` block's view has `raw(SafeHTML)`, and a `SafeHTML` is made only from a string literal: `SafeHTML.trusted("<svg …>")`. `Punch/TrustedHTML` reports anything else, so markup built from data cannot slip in.

## Layouts and navigation

A layout is a component that takes the page's content as a block and writes the document around it:

```crystal source=reference/app/components/layout.cr select=Invoicing::Layout
# app/components/layout.cr
module Invoicing
  # The frame of every page: the document with the app's stylesheet and
  # Turbo, navigation for a signed-in user, and the content in `<main>`,
  # in the request's language (`v.t`). Navigation is chrome, which pages
  # that agents read leave out. A live page passes *morph*, so
  # Turbo morphs it when a change refreshes it.
  struct Layout
    include Punch::Component

    def initialize(@title : String, @actor : Actor? = nil, *, @morph : Bool = false)
    end

    def view(v : Punch::View, &) : Nil
      v.document(@title, stylesheets: {Punch.asset("app.css")}, morph: @morph) do
        if actor = @actor
          t = v.t.layout
          v.chrome do |chrome|
            chrome.header do
              chrome.img src: Punch.asset("images/mark.svg").href, alt: ""
              chrome.nav(aria: {label: t.main}) do
                chrome.a t.invoices, href: Billing::Invoices::IndexPage.href
                chrome.text " "
                chrome.a t.customers, href: Billing::Customers::IndexPage.href
                # Shown to operators only: the link asks the console's policy.
                chrome.text " "
                chrome.a t.ops, href: Punch::Ops::Console::IndexPage.href
              end
              chrome.p do
                chrome.text "#{t.signed_in_as} "
                chrome.strong actor.user.name.to_s
                if organization = actor.organization
                  chrome.text " #{t.in_organization} "
                  chrome.strong organization.name.to_s
                end
                chrome.text " "
                chrome.a t.organizations, href: Accounts::Organizations::IndexPage.href
                chrome.text " "
                chrome.a t.account, href: Accounts::Sessions::IndexPage.href
              end
              chrome.form(Accounts::Sessions::SignOut.action) { |form| form.submit t.sign_out }
            end
          end
        end
        v.main { yield }
      end
    end
  end
end
```

- **`v.document`** writes the doctype, `<html lang>` in the reader's language, the head with the title, the stylesheets, the import map, and Turbo, and the block as the body. It must be the outermost element, once. `morph: true` makes Turbo morph the page when it is refreshed, keeping the scroll position and what the viewer is typing ([Turbo Frames, Streams, and Live Pages](turbo-frames-streams-and-live-pages.md)).
- **`v.chrome`** holds navigation, decoration, and anything else that serves people only. It is written in HTML and left out of the markdown agents read. Put navigation there from the start, and nothing changes when a page becomes agent-readable.
- **Links in navigation ask their policies.** The `ops` link points at the ops console, which only operators may open. On a page that takes the actor, a link to a page the viewer may not open is left out, with its content, so the layout needs no `if operator?`.

A page renders its content inside the layout, as `DetailView` does:

```crystal illustrative="the first lines of DetailView#view, quoted in full above"
v.render Invoicing::Layout.new(title, @actor, morph: true) do
  v.h1 title
  # ...
end
```

App-wide layouts live in `app/components/`, one file each.

## Links between pages

A link is a typed reference to a page, never a hand-written URL:

```crystal illustrative="calls from the reference app's components"
Invoices::ShowPage.href(invoice_id: invoice.id)        # /billing/invoices/42
Invoices::IndexPage.href(after: last.id, limit: 20)    # /billing/invoices?after=42&limit=20
Punch::Href.external("https://example.org/terms")      # another site
```

`href` takes every path parameter and any of the page's other input fields, each of its field's type, checked at compile time. Renaming a page, moving it, or changing its input breaks every link to it at compile time instead of in production:

```crystal fails=spec/fail/href_wrong_type.cr
struct View
  include Punch::Component

  def view(v : Punch::View) : Nil
  end
end

module Notes
  @[Punch::Public]
  struct IndexPage < Punch::Page
    struct Input
      include Punch::Input
      getter page : Int32 = 1
      getter tag : String
    end

    def call(input : Input, db : Punch::Reader) : View
      View.new
    end
  end
end

Notes::IndexPage.href(tag: "a", page: "2")
```

```text
Error: Notes::IndexPage.href: `page` must be Int32, not String
```

A string where a link belongs does not compile either, so there is no way to write `href: "/billing/invoices"` and have it drift from the route:

```text
Error: no overload matches 'Punch::HTMLView#a' with types String, href: String
```

`Href.external` accepts absolute `https`, `http`, `mailto`, and `tel` URLs and raises on anything else, `javascript:` included. For an address from user data, `Href.external?` returns `nil` instead of raising.

Commands are never linked. A change of state is a form, so it is sent with `POST`, carries the cross-site check, and is never triggered by a crawler following a link ([Forms](forms.md)).

## What a page answers when things go wrong

| Situation | Answer |
|---|---|
| `call` returns a failure, such as `NotFound` | Punch's problem page, with the failure's status, message, and code |
| A path parameter that does not parse (`/billing/invoices/abc`) | 404: the address names no page |
| A query parameter that does not parse | 422, listing each field and code |
| No signed-in user where the page needs one | 303 to the app's sign-in page (`Punch::Pages.sign_in` in `app/setup.cr`) |
| The policy refuses | 404 or 403, as for any operation |

A component can be a failure too: a struct that includes `Punch::Component` and has `@[Punch::Status(410)]` renders itself with that status, for a "this link has expired" page with its own design.

## Testing pages and components

A page spec visits the page as a browser does and checks the HTML with CSS selectors:

```crystal source=reference/spec/billing/invoices/show_page_spec.cr select="invoice pages shows a sent invoice with its customer, dates, lines, and total"
# spec/billing/invoices/show_page_spec.cr
# ...

describe "invoice pages" do
  it "shows a sent invoice with its customer, dates, lines, and total" do
    client = AccountsSpec.client
    id = invoice(client, customer(client), [{description: "Consulting <on site>", quantity: 3, unit_price: 12_500}, {description: "Travel", quantity: 1, unit_price: 4_550}], send: true)
    page = client.visit(Billing::Invoices::ShowPage.href(invoice_id: id))
    page.status.should eq(HTTP::Status::OK)
    page.should have_element("title", text: "Invoice INV-000001")
    page.should have_element("main > h1", text: "Invoice INV-000001")
    page.should have_element("dl.facts dd", text: "Ada Lovelace")
    page.should have_element("dl.facts dd", text: "sent")
    page.should have_element("dl.facts dd time[datetime]", count: 2)
    page.should have_element("table.lines tbody tr", count: 2)
    page.should have_element("table.lines td", text: "Consulting <on site>")
    page.should have_element("table.lines caption", text: "2 lines")
    page.should have_element("table.lines td", text: "€125.00")
    page.should have_element("table.lines tfoot td", text: "€420.50")
    page.should have_element("header nav a[href='/billing/invoices']", text: "Invoices")
    page.should have_element("header strong", text: "Ingrid Iversen")
  end

  # ...
end

# ...
```

- `client.visit(href)` asks with a browser's `Accept`. HTML responses are not checked against `openapi.json`; JSON responses of the same client are.
- `have_element(selector, text:, count:)` finds elements by type, class, ID, attribute, and combinator. `text` is compared with whitespace collapsed, or matched by a `Regex`.
- The spec checks escaping on the way: the description `Consulting <on site>` comes back as text.

A component can be rendered alone, without a request:

```crystal illustrative="a spec shape; the reference app checks LinesTable through its pages"
Punch::Spec.render(LinesTable.new(detail)).should have_element("td", text: "€125.00")
```

Every page and component that a spec renders also has its nesting checked, so render each at least once.

## Common mistakes

**Putting rules in the page.** A page that checks `if detail.status.draft?` before deciding to send something has taken a rule from the model. Pages decide what to show; Commands and models decide what may happen. Showing the Send button only for a draft is fine; refusing to send a sent invoice is `Send`'s job, and it does it.

**A string for a link.** `v.a "Invoices", href: "/billing/invoices"` does not compile. Use `Billing::Invoices::IndexPage.href`.

**A sentence as a literal.** `v.p "This draft gets its number when it is sent."` compiles, and is wrong in every language but one. Add the text to `config/locales/en.yml` and write `v.p t.draft_note`. No tool checks this yet; it is in the app's `AGENTS.md` list of rules to keep by hand.

**Formatting with `to_s`.** `v.td total.to_s` shows minor units, `30000`, to a reader who expects `€300.00`. Use `v.locale.money(total, currency)`, `v.locale.date`, and `v.locale.number`.

**An inline style or event handler.** `v.div(style: "color: red")` and `v.button(onclick: …)` do not compile. Use a class and the stylesheet, or a JavaScript module.

**Navigation outside `chrome`.** Links in `v.nav` directly under `v.document` show up in the markdown agents read, on every page. Put navigation in `v.chrome`.

**A list item outside a list.** `v.ul { v.p "None" }` raises `NestingError` when the component renders. Write the empty state outside the list, or as an `li`.

## Limits

- Images are written with `v.img`; there is no image processing, resizing, or upload.
- No partial page caching. A page renders on every request; cache the expensive results instead ([Caching](caching.md)).
- No client-side rendering. Interactivity comes from Turbo and small JavaScript modules.

## Rules

- Keep pages thin: call operations, render components; rules stay in operations and models. ([A page calls operations and renders components](#a-page-calls-operations-and-renders-components))
- One page per file in `pages/`, named `…Page`; components only it renders in its file, shared ones in `components.cr`. ([Where pages live, and their routes](#where-pages-live-and-their-routes))
- Link with `Page.href`, never with a string. ([Links between pages](#links-between-pages))
- Never link to a Command; send a form. ([Links between pages](#links-between-pages))
- Put navigation and decoration in `v.chrome`. ([Layouts and navigation](#layouts-and-navigation))
- No inline scripts or styles; no markup from data. ([Why no template language](#why-no-template-language))
- Text from the locale files, numbers and dates through `v.locale`. ([Components](#components))
- Render every page and component in a spec at least once. ([Testing pages and components](#testing-pages-and-components))

## Reference

- [Views and pages](../views.md): pages, components, every element and attribute, nesting, and links.
- [Assets](../assets.md): stylesheets, images, and scripts in `v.document`.
- [Authorization](../authorization.md): links and forms that ask the policy.
- [Testing](../testing.md): `client.visit`, `have_element`, and `Punch::Spec.render`.
