# Pages for Agents

Applies to Punch 0.1.

A Punch app has three surfaces: HTML for people, JSON for programs, and markdown for agents. An agent, such as a coding assistant or an AI working for one of the app's users, reads the same pages a person does, as markdown, and acts through the same forms, which appear as action blocks at the end of each page. There is no separate agent API to build or keep in step. An agent sees and does exactly what its user may, because every page and every form runs the same policies as in the browser.

After reading this guide, you will know:

- how to let agents read a page as markdown;
- what an agent sees of a page, and of its forms;
- how an agent acts on a page, and what it gets back;
- how an agent finds its way in from `/llms.txt`;
- how users give agents tokens safely;
- how to test a page the way an agent uses it.

Before you start: you know [Forms](forms.md) and [Authentication](authentication.md). The examples come from the reference app, whose invoice list and invoice pages are agent-readable.

## Opting a page in

A page that agents may read says so with an annotation. The reference app's invoice page:

```crystal illustrative="the annotations of Billing::Invoices::ShowPage, quoted in Pages and Components"
@[Punch::AgentReadable]
@[Punch::Authorize(ByRole)]
@[Punch::Forms(Send, Destroy)]
struct ShowPage < Punch::Page
```

Nothing else changes. The page renders the same components, and Punch writes them as markdown when an agent asks. A page without the annotation answers HTML whatever the client asks for, so nothing reaches the markdown surface by accident.

An agent asks in one of two ways:

- **`Accept: text/markdown`** on the page's own path. The answer carries `Content-Type: text/markdown; charset=utf-8; variant=GFM` and `Vary: Accept`.
- **The `.md` URL**: `/billing/invoices/3.md`, for agents that do not send `Accept`. It answers markdown whatever the headers say.

The HTML of an agent-readable page names its markdown in the head, `<link rel="alternate" type="text/markdown" href="/billing/invoices/3.md">`, so an agent that lands on the HTML finds its way.

## What an agent reads

Here is a draft's page as the reference app answers it:

```console
$ curl -i http://localhost:3000/billing/invoices/3.md -H "Authorization: Bearer inv_…"
HTTP/1.1 200 OK
Content-Type: text/markdown; charset=utf-8; variant=GFM
Content-Language: en

# Draft 3

- Customer: Ada Lovelace
- Status: draft

A draft can still change; it gets its number when it is sent.

1 line

| \# | Description | Quantity | Unit price | Amount |
|---|---|---|---|---|
| 1 | Workshop | 1 | €1,200.00 | €1,200.00 |
| Total | **€1,200.00** | | | |

## Actions

Send an action to its address with its fields as a form body (`application/x-www-form-urlencoded`), or as query parameters for a `GET`. Success answers `303 See Other` with the next page in `Location`; a refusal answers with this page again, the errors in the action.

### Send invoice

`POST /billing/invoices/3/send.md` · operation `billing.invoices.send` · not idempotent · at most 30 per 1 h per user

- `_idempotency_key` = `766i5UBC5-S7LRnKAJx1Cgh_` (send it as it is, also when you retry)

### Delete draft

`DELETE /billing/invoices/3.md` · operation `billing.invoices.destroy` · idempotent · destructive
```

The same components wrote the HTML page. In markdown:

- **Headings, paragraphs, lists, and tables** become their markdown forms; a definition list becomes a list of `term: description`.
- **Containers** such as `div`, `main`, and frames are transparent, and an image is its alt text.
- **Chrome is left out**: the layout's navigation, the user's name, the sign-out button. That is why navigation belongs in `v.chrome` from the start ([Pages and Components](pages-and-components.md#layouts-and-navigation)).
- **Links to agent-readable pages** lead to their `.md` URLs, so an agent stays on markdown as it follows them.
- **Text stays text.** No value from the database can turn into a link, emphasis, or HTML in the markdown, which is checked against hostile strings.

## Action blocks: forms for agents

Markdown has no forms, so each form on the page becomes an action block under `## Actions`. Nothing is declared for this: the block is generated from the form as the page renders it.

- **The title** is the form's `aria` label, else its button's text. Give forms that repeat on a page, such as a delete button per row, a label that says which.
- **The address** is the Command's real method and path with `.md`, and the operation ID as in `openapi.json`.
- **Safety hints** follow from the method: `GET` is safe, `PUT` and `DELETE` are idempotent, `DELETE` is destructive, `POST` and `PATCH` are not idempotent.
- **Rate limits** are stated, so an agent can pace itself.
- **The fields** are exactly the controls the form writes, by the names it sends: each with its label, what it takes (text, email address, whole number, one of a list of values), whether it is required, its bounds, its hint, and its current value. Hidden fields, such as the idempotency key, are values to send as they are. Passwords are never shown.
- **An action appears only where its form does.** The Send block is on a draft's page, not on a sent invoice's, and not for a member whose role may not send. The page's state and the viewer's policy decide, exactly as for the button in HTML.

The phrases follow the reader's language; field names, values, operation IDs, and codes never change, because agents copy them.

## Acting

An agent acts by sending the block's fields as a form body to its address, with its token:

```console
$ curl -i -X POST http://localhost:3000/billing/invoices/3/send.md \
    -H "Authorization: Bearer inv_…" --data-urlencode "_idempotency_key=766i5UBC5-S7LRnKAJx1Cgh_"
HTTP/1.1 303 See Other
Location: /billing/invoices/3.md
```

The Command ran as it would for a browser or a JSON client, and the answer leads to the next page, in markdown. Sending the same action again with the same key replays the first answer:

```console
$ curl -i -X POST http://localhost:3000/billing/invoices/3/send.md \
    -H "Authorization: Bearer inv_…" --data-urlencode "_idempotency_key=766i5UBC5-S7LRnKAJx1Cgh_"
HTTP/1.1 303 See Other
Idempotent-Replayed: true
Location: /billing/invoices/3.md
```

| Answer | When |
|---|---|
| `303 See Other` to the next page | the Command succeeded |
| the page again, as markdown, with the refusal's status | a rejection (422), a failure result (409, …), or a refusal by policy (403): field errors under their fields, others at the top of the action |
| the page with the problems at the top of `## Actions` | the refused form is no longer on the page, such as a Send that failed because the invoice was sent in between |

An agent without a token gets a 401 in markdown, never the redirect to the sign-in page a browser gets.

## Discovery: `/llms.txt`

An agent given only the app's address starts at `/llms.txt`, a page in the [llmstxt.org](https://llmstxt.org) format that Punch writes from the route table. It lists the agent-readable pages marked as entry points:

```crystal illustrative="the annotations of Billing::Invoices::IndexPage"
@[Punch::AgentReadable(entry: true)]
@[Punch::Authorize(Accounts::Organizations::Current)]
struct IndexPage < Punch::Page
```

and explains, in the reader's language, how to read, authenticate, act, retry, and wait out a rate limit:

```console
$ curl http://localhost:3000/llms.txt
# Invoicing

> Invoices and customers of the organization you work in. Draft an invoice, send it, and see when it is paid.

Each page listed below is markdown at its `.md` address; the same path without `.md` answers markdown to `Accept: text/markdown`. Links in a page lead to the markdown of the pages they open.

Send the user's API token as `Authorization: Bearer <token>`. It acts as the user: you see and may do what they may, nothing more. A page that needs a user answers `401` without one.

A page ends with what you can do there, under `## Actions`. Send an action's fields as a form body (`application/x-www-form-urlencoded`) to the address it names. Success answers `303 See Other` with the next page in `Location`; a refusal answers with the page again, the problems in the action.

If an answer is lost, send the action again with the same idempotency key: its `_idempotency_key` field, or the `Idempotency-Key` header you sent. You get the first answer back, and the action does not run twice. A new key runs it again. Keys are kept for 24 hours.

`429 Too Many Requests` means: wait the seconds in `Retry-After`, then send again.

## Pages

- [Invoices](/billing/invoices.md): the organization's invoices, found by number or customer and narrowed by state; each links to its page, where you send or delete it

## Optional

- [OpenAPI](/openapi.json): the JSON API's contract, for programs
```

- **Entry points** are pages an agent can open from their path alone, so an entry point with a path parameter or a required field does not compile. Other pages are reached by links.
- **The app's text** comes from the locale files: `llms.title`, `llms.summary`, and per entry point `llms.pages.<operation id>.title` and `.description`. Describe what an agent can do there, as the invoice list's description does.
- **It is public.** It lists paths, as `openapi.json` does; every page still answers by its policy.

## Tokens for agents

An agent authenticates with an API token of the user it works for, sent as `Authorization: Bearer …`. The token acts as that user, with all of their permissions and nothing more; revoking it or letting it expire is the control.

Users make tokens themselves, on a page `punch new` ships in `app/accounts/tokens/`:

- **Making one needs a recent sign-in with a password.** A token can never qualify, so a token cannot make another token, and a session left open for a while asks for the password again.
- **The token is shown once**, right after it is made, carried to the next page in a short-lived encrypted cookie and never stored in clear.
- **Every token expires**, after 7 to 366 days, and the page shows when each was last used.

See [Authentication](authentication.md#sessions-and-api-tokens).

## What does not compile on an agent-readable page

An agent-readable page renders through a plain `Punch::View`, which has no `raw`. A component that writes raw HTML outside a chrome block fails to compile on such a page:

```text
Error: undefined method 'raw' for Punch::View
```

A component that declares it needs an HTML view says so too:

```text
Error: Chart takes a Punch::HTMLView, so it may write raw HTML, and cannot render where only semantic elements are allowed
```

Move the markup into a chrome block, or write it with elements. Pages for people only are not affected.

## Testing pages as an agent

A spec reads a page as an agent does and acts through its blocks:

```crystal source=reference/spec/journeys/agents_spec.cr select="agents find a draft through the list's filter and send it from its page"
# spec/journeys/agents_spec.cr
# ...

describe "agents" do
  it "find a draft through the list's filter and send it from its page" do
    client, headers, customer = agent
    id = draft(client, headers, customer)
    list = client.read(Billing::Invoices::IndexPage.href, headers: headers)
    list.content_type.should eq("text/markdown")
    list.should have_action("Filter", operation: "billing.invoices.index_page")
    drafts = client.act(list, "Filter", {"filter[status]" => "draft"}, headers: headers)
    drafts.body.should contain("(/billing/invoices/#{id}.md)")

    page = client.read(Billing::Invoices::ShowPage.href(invoice_id: id), headers: headers)
    page.should have_action("Send invoice", operation: "billing.invoices.send")
    page.should have_action("Delete draft", operation: "billing.invoices.destroy")
    send = Punch::Spec.actions(page.body).find!(&.title.==("Send invoice"))
    # Send requires an idempotency key: the block carries one to reuse.
    send.fixed.keys.should eq([Punch::IdempotencyKeys::FIELD])
    send.text.should contain("at most 30 per 1 h per user")

    sent = client.act(page, "Send invoice", headers: headers)
    sent.should redirect_to(Billing::Invoices::ShowPage.href(invoice_id: id).markdown_url)
    after = client.read(sent.headers["Location"], headers: headers)
    after.should_not have_action("Send invoice")
    after.body.should contain("- Status: sent")
  end

  # ...
end
```

- `client.read(href, headers:)` asks with `Accept: text/markdown`; `response.body` is the markdown.
- `have_action(title, operation:)` finds an action block, and `Punch::Spec.actions(body)` parses them all.
- `client.act(page, title, fields, headers:)` sends an action as an agent would: the block's fixed fields, such as the idempotency key, plus the given ones. It fails the example for a field the block does not name, because an agent could not have known it.

The reference app also checks an agent that starts from `/llms.txt` with nothing but the base URL and a token, which is the test that the surface is complete.

## Common mistakes

**Navigation outside `chrome`.** Every agent-readable page then starts with the app's menu in markdown. Put navigation in `v.chrome`.

**Forms without a title an agent can tell apart.** A list with a "Delete" button per row gives the agent ten actions called "Delete". Give each form `aria: {label: t.delete_invoice(number: …)}`.

**A lazy frame with nothing inside.** Lazy frames are empty in markdown. Put a link inside.

**An entry point that needs a parameter.** `@[Punch::AgentReadable(entry: true)]` on a page with a path parameter does not compile:

```text
Error: Reports::ShowPage is an entry point in /llms.txt, so an agent opens it from its path alone. Its path parameter `report_id` cannot be filled from there
```

Make the list the entry point and link to the page from it.

**Giving agents a separate API.** A set of JSON endpoints "for the agent" duplicates what pages already offer, and drifts. Mark the pages agent-readable; programs that want JSON already have every operation.

**Scopes on tokens.** There are none: a token can do what its user can. Give an agent a token from a user whose role fits the agent's job.

## Limits

- No token scopes: a token acts as its user.
- Inputs of one of several shapes cannot be expressed as action blocks yet.
- A textarea's line breaks are shown on one line in an action's current value.
- No annotation yet for a `POST` that destroys, so such an action is marked "not idempotent" only.

## Rules

- Mark a page `@[Punch::AgentReadable]` only when agents should read it; mark the lists agents start from as entry points. ([Opting a page in](#opting-a-page-in), [Discovery: `/llms.txt`](#discovery-llmstxt))
- Keep navigation and decoration in `v.chrome`. ([What an agent reads](#what-an-agent-reads))
- Give every form a title an agent can tell apart. ([Action blocks: forms for agents](#action-blocks-forms-for-agents))
- Never build a separate API for agents; the pages and their forms are the agent surface. ([Common mistakes](#common-mistakes))
- Agents authenticate with their user's token; there is no other identity for them. ([Tokens for agents](#tokens-for-agents))
- Test agent-readable pages with `client.read` and `client.act`. ([Testing pages as an agent](#testing-pages-as-an-agent))

## Reference

- [Views and pages](../views.md#pages-agents-read): the markdown surface, discovery, and action blocks in full.
- [Authentication](../auth.md#tokens-for-agents): tokens for agents, and showing a new token once.
- [Testing](../testing.md): `client.read`, `client.act`, and `have_action`.
