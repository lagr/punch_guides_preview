# Turbo Frames, Streams, and Live Pages

Applies to Punch 0.1.

Punch pages are server-rendered HTML, made lively by Turbo, which every page loads. Turbo Drive makes links and forms feel immediate without anything from you. Beyond that, three tools change part of a page instead of all of it: a frame loads or replaces one region, a stream changes elements of a page already shown, and a live page receives streams over server-sent events whenever a Command commits something it shows. All three reuse the page's own components, so there is never a second template for the same thing.

After reading this guide, you will know:

- how to load or replace part of a page with a Turbo Frame;
- how to answer a form with a Turbo Stream instead of a redirect;
- how to keep a page current for everyone looking at it, and decide per viewer what changes;
- why a live page never misses a change, and when it refreshes instead;
- what live pages cannot do yet;
- how to test frames, streams, and live pages.

Before you start: you know [Forms](forms.md) and [Domain Events](domain-events.md), since live pages are driven by events. The examples come from the reference app's customer and invoice pages.

## Turbo Drive comes first

Every Punch page loads Turbo, so links and forms already avoid full page loads: Turbo fetches the next page and swaps its body in. Forms follow Turbo's conventions without anything from you: a successful Command answers 303 to the next page, a refused one answers 422 with the page again, shown in place ([Forms](forms.md#after-sending-303-or-the-page-again)).

Start there. Most pages need nothing more. Reach for the tools below when a page has a region that loads on its own, a form whose answer should change only part of the page, or content that changes while someone looks at it.

## Frames: one region of a page

`v.frame(id) { … }` writes a `<turbo-frame>`: a region that links and forms inside it replace on their own, or that loads itself from another page. The reference app's customer edit page shows the customer's invoices below the form, loaded lazily from the invoice list:

```crystal source=reference/app/billing/customers/pages/update_page.cr select=Billing::Customers::EditView
# app/billing/customers/pages/update_page.cr
module Billing::Customers
  @[Punch::Status(200)]
  struct EditView
    include Punch::Component

    def initialize(@customer : Customer, @actor : Invoicing::Actor)
    end

    def view(v : Punch::View) : Nil
      t = v.t.customers
      title = @customer.display_name
      v.render Invoicing::Layout.new(title, @actor) do
        v.h1 title
        values = {name: @customer.name, company_name: @customer.company_name, email: @customer.email, locale: @customer.locale}
        v.form(Update.action(customer_id: @customer.id), values: values) do |form|
          form.field :name, label: t.contact_person
          form.field :company_name
          form.field :email
          form.field :locale, as: :select, options: Invoicing::Languages.options, label: t.mail_language, hint: t.mail_language_hint
          form.submit t.save
        end
        v.h2 t.invoices
        # Loaded by Turbo once it is in view; without it, the link.
        v.frame(@customer.id, "invoices", src: Invoices::IndexPage.href(filter: Invoices::Queries::Page::Filter.new(customer_id: @customer.id)), loading: "lazy", target: "_top") do
          v.p { v.a t.invoices_of(name: title), href: Invoices::IndexPage.href(filter: Invoices::Queries::Page::Filter.new(customer_id: @customer.id)) }
        end
      end
    end
  end

  # ...
end
```

- **The frame's ID comes from a typed ID.** `v.frame(@customer.id, "invoices")` writes `id="invoices_customers_7"`. `Punch.dom_id(id, "invoices")` gives the same string anywhere else, so the page that answers the frame's request writes a frame with the same ID, and Turbo takes its content from there.
- **`src:` with `loading: "lazy"`** loads the frame once it scrolls into view, from the invoice list filtered to this customer. The list page wraps its table in the same frame, so one page serves both a full visit and the frame.
- **`target: "_top"`** makes links inside the frame navigate the whole page, so clicking an invoice opens it, rather than replacing only the frame.
- **The link inside** is what a browser without Turbo, or an agent reading markdown, sees instead. A frame that loads lazily is empty in markdown, so always give it a link.

A frame works as plain HTML without Turbo, and a spec can check both sides:

```crystal source=reference/spec/billing/customers/update_page_spec.cr select="customer forms loads the customer's invoices into a frame with the same ID"
# spec/billing/customers/update_page_spec.cr
# ...

describe "customer forms" do
  # ...

  it "loads the customer's invoices into a frame with the same ID" do
    client = AccountsSpec.client
    id = customer(client)
    draft(client, id)
    frame_id = Punch.dom_id(id, "invoices")
    edit = client.visit(Billing::Customers::UpdatePage.href(customer_id: id))
    edit.should have_element("turbo-frame##{frame_id}[loading=lazy][src='#{Billing::Invoices::IndexPage.href(filter: Billing::Invoices::Queries::Page::Filter.new(customer_id: id))}'] a")
    list = client.visit(Billing::Invoices::IndexPage.href(filter: Billing::Invoices::Queries::Page::Filter.new(customer_id: id)))
    list.should have_element("turbo-frame##{frame_id}[target=_top] tbody tr", count: 1)
  end
end
```

## Streams: changes to a page already shown

A `Punch::TurboStream` is a list of changes to elements of a page that is already open. Each change carries a component, rendered as HTML, so the row that is part of the page is sent as it is:

```crystal illustrative="every kind of action, with components from the reference app"
Punch::TurboStream.build do |stream|
  stream.prepend "invoices", ListRow.new(summary)           # first child of #invoices
  stream.replace invoice.id, ListRow.new(summary)           # the element #invoices_42 itself
  stream.update invoice.id, ListRow.new(summary), morph: true
  stream.remove Punch.dom_id(line.id, "lines")
  stream.refresh                                            # Turbo fetches the page again
end
```

- **Actions:** `append` and `prepend` (inside the target), `replace` (the target itself), `update` (what the target holds), `before` and `after` (siblings), `remove`, and `refresh`.
- **`morph: true`** on `replace` and `update` changes only what differs, keeping focus and what the viewer is typing.
- **A target is a DOM ID:** a typed ID, written by `Punch.dom_id` as frames and rows write it (`invoices_42`), or a string for an element the page names itself (`v.tbody(id: "invoices")`).
- **Content is a component**, never the layout: a stream carries parts of a page, and rendering a document in it raises.

Forms and links inside a stream's content ask their policies for the viewer, as on a page.

## Answering a form with a stream

A page that holds a Command's form can answer it with a stream instead of the redirect, per result type, next to `redirect`:

```crystal illustrative="a list page that adds a new customer's row in place; the reference app redirects instead"
@[Punch::Forms(Create)]
struct IndexPage < Punch::Page
  # ...

  def self.stream(result : Created) : Punch::TurboStream?
    Punch::TurboStream.build { |stream| stream.prepend "customers", CustomerRow.new(result.customer) }
  end
end
```

- Only a request whose `Accept` names `text/vnd.turbo-stream.html`, as Turbo sends with a form, gets the stream. A browser without Turbo, and a result the page has no `stream` for, get the 303 as before.
- Refusals are unchanged: the page again, with the errors, which Turbo shows in place.
- Build the stream from the result. Data the result does not carry belongs in the result, or the stream asks for `refresh`.

The redirect is the simpler default, and with `v.document(…, morph: true)` in the layout, a redirect back to the same page is already smooth. Write a `stream` when the page would lose state the viewer cares about, such as a long list scrolled far down.

## Live pages: changes made elsewhere

A page that defines `live` updates while it is open. When a Command commits an event the page takes, the page decides for each open viewer what that event changes on their screen, and the answer reaches the browser as stream actions over server-sent events.

The reference app's invoice page refreshes itself when that invoice changes:

```crystal source=reference/app/billing/invoices/pages/show_page.cr select=Billing::Invoices::ShowPage#live
# app/billing/invoices/pages/show_page.cr
module Billing::Invoices
  # ...

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
    # ...

    def live(event : InvoiceChanged, input : Input, db : Punch::Reader, actor : Invoicing::Actor, clock : Punch::Clock) : Punch::TurboStream?
      Punch::TurboStream.build(&.refresh) if event.invoice_id == input.invoice_id
    end

    # ...
  end
end
```

`InvoiceChanged` is a module that every event changing an existing invoice includes (`app/billing/events.cr`), so one `live` hears sends, payments, revisions, and deletions. `refresh` asks Turbo to fetch the page again; with `morph: true` in the layout, Turbo morphs it, keeping the scroll position.

The invoice list is more careful, because it is shown with filters and pages, and a change can mean different things for different viewers:

```crystal source=reference/app/billing/invoices/pages/index_page.cr select=Billing::Invoices::IndexPage#live,Billing::Invoices::IndexPage#changed
# app/billing/invoices/pages/index_page.cr
module Billing::Invoices
  # ...

  # The search screen: the invoices of the organization the actor works
  # in, a page at a time, found by number or customer and narrowed by
  # state. `GET /billing/invoices` in a browser.
  #
  # Open in a browser, it follows the organization's invoices (`live`,
  # lib/punch/docs/views.md): a changed row is replaced where it is shown, a row that no
  # longer matches the filter is removed, and where an invoice may have
  # come into the list (a new draft, or one that now matches the state or
  # the search), Turbo refreshes the page by morphing it.
  @[Punch::AgentReadable(entry: true)]
  @[Punch::Authorize(Accounts::Organizations::Current)]
  struct IndexPage < Punch::Page
    # ...

    # New drafts go to the end of the list, on whichever page that is.
    def live(event : InvoiceDrafted, input : Input, db : Punch::Reader, actor : Invoicing::Actor, clock : Punch::Clock) : Punch::TurboStream?
      return unless event.organization_id == actor.organization_id

      Punch::TurboStream.build(&.refresh) if Index.listed(db, event.invoice_id, input, actor, clock)
    end

    # Revising a draft changes neither its state nor what the search finds.
    def live(event : DraftRevised, input : Input, db : Punch::Reader, actor : Invoicing::Actor, clock : Punch::Clock) : Punch::TurboStream?
      changed(event.invoice_id, input, db, actor, clock, entering: false)
    end

    # Sending changes the state and gives the number the search finds.
    def live(event : InvoiceSent, input : Input, db : Punch::Reader, actor : Invoicing::Actor, clock : Punch::Clock) : Punch::TurboStream?
      filter = input.filter
      changed(event.invoice_id, input, db, actor, clock, entering: !filter.status.nil? || !filter.q.nil?)
    end

    # A payment changes the state only.
    def live(event : InvoicePaid, input : Input, db : Punch::Reader, actor : Invoicing::Actor, clock : Punch::Clock) : Punch::TurboStream?
      changed(event.invoice_id, input, db, actor, clock, entering: !input.filter.status.nil?)
    end

    def live(event : DraftDeleted, input : Input, db : Punch::Reader, actor : Invoicing::Actor, clock : Punch::Clock) : Punch::TurboStream?
      Punch::TurboStream.build(&.remove(event.invoice_id)) if event.organization_id == actor.organization_id
    end

    # What a change to invoice *id* means for the viewer's list: the row
    # again if it still matches, a refresh when it may not have been
    # listed before the change (*entering*), and its removal when it no
    # longer matches. Nothing for another organization's invoice.
    private def changed(id : ID, input : Input, db : Punch::Reader, actor : Invoicing::Actor, clock : Punch::Clock, *, entering : Bool) : Punch::TurboStream?
      if summary = Index.listed(db, id, input, actor, clock)
        Punch::TurboStream.build { |stream| entering ? stream.refresh : stream.replace(id, ListRow.new(summary), morph: true) }
      elsif Queries.organization(db, invoice_id: id).try(&.organization_id) == actor.organization_id
        Punch::TurboStream.build(&.remove(id))
      end
    end
  end
end
```

One `live` per event type, each taking the event and then what `call` takes:

- **A new draft** refreshes the list only if the viewer's filter would show it.
- **A revised draft** replaces its row, morphed, where it is shown.
- **A sent or paid invoice** replaces its row, or refreshes the list when the change may bring it into a filtered view (a search that now matches its new number), or removes it when it no longer matches.
- **A deleted draft** removes its row.
- **Another organization's invoice** changes nothing: `nil` means "nothing for this viewer".

Whether a changed invoice still matches is decided by `Index.listed`, which runs the list's own query, with the viewer's filter, for that one ID. The page never re-implements its filter in Crystal.

## How a live page stays correct

A live page needs no channel names, no subscriptions, and no signed stream tokens:

- **The stream is the page's own URL.** A live page ends its body with `<turbo-stream-source src="/billing/invoices?…&_live=…">`, and Turbo opens it with `EventSource`. The same authentication and policy run as for the page; a refusal answers 401, 403, or 404, and the browser gives up.
- **Every event is decided for each viewer.** For every event the page takes, Punch runs the policy and `live` again in a read transaction of the stream's own, for that viewer, in their language. A viewer who lost access gets no more events: the stream ends.
- **Nothing is missed.** `_live` records where the event hub stood right before the page was rendered. The stream first replays the events after it, then follows. A browser that reconnects sends the last event's ID and catches up the same way. When the hub no longer holds those events, the stream sends `refresh` and Turbo fetches the page.
- **An event may arrive twice**, around the moment the page was rendered. `replace`, `update`, `remove`, and `refresh` are safe to repeat, and Turbo removes an element with the same ID before an `append` or `prepend`. Write `live` with actions that are safe to repeat.

> [!IMPORTANT]
> `live` must scope what it builds to the viewer. The invoice list checks `event.organization_id == actor.organization_id` before reading, and `Index.listed` runs with the viewer's organization. `refresh` is always safe, since the refreshed page runs its own policy; a row built in `live` is shown as it is.

## Keeping `live` cheap

`live` runs once per open stream that takes the event, each in its own read transaction: a hundred viewers of a list cost a hundred renders per change.

- **Return early on what the event says**, before reading: the organization, the page's input.
- **Put what `live` needs in the event.** `DraftDeleted` carries the organization, because nothing is left to look it up.
- **Prefer `refresh` for changes too wide to describe.** It costs each viewer one page load, which Turbo morphs.

A `live` that raises skips that event for that stream; the error is recorded in the ops console and the stream goes on.

## Testing live pages

`client.live(href)` opens a page's stream in a spec; `live.next` waits for the next message's actions:

```crystal source=reference/spec/billing/invoices/index_page_spec.cr select="the invoice list, open in a browser replaces a revised draft's row, and removes a deleted one"
# spec/billing/invoices/index_page_spec.cr
# ...

describe "the invoice list, open in a browser" do
  # ...

  it "replaces a revised draft's row, and removes a deleted one" do
    watching, acting, customer, _ = windows
    id = draft(acting, customer)
    live = watching.live(list)
    acting.patch(Billing::Invoices::Update.path(invoice_id: id), json: {lines: [{description: "Workshop", quantity: 1, unit_price: 99_000}]})
      .status.should eq(HTTP::Status::OK)
    live.next.should have_stream_action(:replace, id, containing: "td", text: "€990.00")
    acting.delete(Billing::Invoices::Destroy.path(invoice_id: id)).status.should eq(HTTP::Status::NO_CONTENT)
    live.next.should have_stream_action(:remove, id)
    live.close
  end

  # ...
end

# ...
```

- `have_stream_action(:replace, id, containing: "td", text: "…")` finds an action by its target and checks an element in its content.
- `live.next?` returns `nil` when no message comes, for checking that a viewer hears nothing, such as another organization's change.
- `live.ended?` tells whether the server ended the stream, and `client.live(href, last_event_id: …)` reconnects.
- `client.submit(action, values, stream: true)` sends a form with Turbo's `Accept`, for a page that answers with a stream.

## Common mistakes

**A frame whose page has no frame with that ID.** Turbo shows "Content missing" when the response has no `<turbo-frame>` with the same ID. Write the frame with `v.frame(id, prefix)` on both pages, from the same typed ID.

**A lazy frame without a link inside.** Browsers without Turbo, and agents reading markdown, see an empty region. Put a link to the frame's page inside it.

**Rendering the layout in a stream.** `stream.replace id, DetailView.new(…)` with a view that renders the layout raises: a stream carries parts of a page. Render the component inside the layout, such as the row.

**A `live` that trusts the event.** Building a row from the event's ID without checking that the viewer may see it shows another organization's data. Scope by the viewer, or return `refresh`.

**Heavy work in `live`.** A `live` that runs the full list query for every event and every viewer turns one change into a burst of queries. Return early, and check one row by ID.

**Expecting a separate jobs process to update pages.** A payment processed by `bin/app jobs` in its own process does not reach the streams of `serve`. Run jobs inside `serve` (the default) while you rely on live pages.

## Limits

- **One process.** Events committed in another process, a separate `jobs` process among them, do not reach this process's streams.
- **Deploys.** On shutdown every stream closes; browsers reconnect to the new process and refresh.
- **Connections.** Each stream holds a connection and a fiber, at most 1000 per process (`Punch::Live.max_streams`); one more is answered 503, and that page stays without live updates until it is loaded again. Browsers allow six HTTP/1.1 connections per host, so many tabs need a reverse proxy that speaks HTTP/2 to browsers.
- **An open page does not keep its session alive**; only requests do.
- A page that takes `Punch::Auth::Reveal` cannot be live.

## Rules

- Start with Turbo Drive and redirects; add frames, streams, and `live` only where a region changes on its own. ([Turbo Drive comes first](#turbo-drive-comes-first))
- Make frame and target IDs from typed IDs with `v.frame` and `Punch.dom_id`. ([Frames: one region of a page](#frames-one-region-of-a-page))
- Give every lazy frame a link inside. ([Frames: one region of a page](#frames-one-region-of-a-page))
- Send components in streams, never the layout. ([Streams: changes to a page already shown](#streams-changes-to-a-page-already-shown))
- Scope everything `live` builds to the viewer; prefer `refresh` when unsure. ([How a live page stays correct](#how-a-live-page-stays-correct))
- Use actions that are safe to repeat. ([How a live page stays correct](#how-a-live-page-stays-correct))
- Return early in `live`, and put what it needs in the event. ([Keeping `live` cheap](#keeping-live-cheap))

## Reference

- [Views and pages](../views.md#turbo-frames): frames, streams, streams after forms, and live pages in full.
- [Domain events](../events.md): the events live pages take.
- [Testing](../testing.md): `client.live` and `have_stream_action`.
