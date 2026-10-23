# Authorization and Roles

Applies to Punch 0.1.

Authentication says who is asking; authorization decides what they may do. In Punch, every operation states who may run it, and the app does not compile until each one does. The decision runs inside the operation's transaction, a Command cannot run without proof that it was allowed, and pages leave out the buttons and links the viewer may not use, by asking the same rule.

After reading this guide, you will know:

- how to decide who may run an operation, and where rules about the request's body belong instead;
- why a policy answers 404 for some refusals and 403 for others;
- how to scope a list to what the viewer may see;
- how to model roles per organization, and give them to users;
- why you never hide a button by hand.

Before you start: you know how a request gets its actor ([Authentication](authentication.md)). The examples come from the reference app, where customers and invoices belong to organizations, and users hold roles in them.

## Every operation says who may run it

An operation carries one of three annotations:

| Annotation | Means |
|---|---|
| `@[Punch::Authorize(Policy)]` | `Policy.check` decides; `call` takes the actor |
| `@[Punch::Authorize(Punch::SignedIn)]` | any signed-in user |
| `@[Punch::Public]` | anyone: sign-in, sign-up, a shared link |

An operation with neither does not compile, and `punch check` confirms it for every route (its `policies` step). There is no default that lets an operation through because someone forgot it. `punch routes --json` shows each route's policy.

## Policies

A policy is a type that includes `Punch::Policy` and defines `self.check`. It receives the actor, the database, and the operation's path parameters, and returns a decision. The reference app's invoices are decided by role:

```crystal source=reference/app/billing/invoices/policies.cr
# app/billing/invoices/policies.cr
module Billing::Invoices
  # Whoever holds a role on the invoice's organization that permits the
  # operation (`Accounts::Roles`): a member sees an invoice but may not
  # send it (403), and to anyone without a role there it does not exist
  # (404).
  struct ByRole
    include Punch::Policy

    def self.check(actor : Invoicing::Actor, db : Punch::Reader | Punch::Writer, operation : Punch::Operation.class, *,
                   invoice_id : ID) : Punch::Policy::Decision
      organization = Queries.organization(db, invoice_id: invoice_id) || return not_found
      Punch::Roles.check(db, actor.user.id, operation, on: organization.organization_id)
    end
  end
end
```

| Decision | Answer | When |
|---|---|---|
| `allow` | the operation runs | |
| `forbid` | 403 `forbidden` | the actor may know the resource exists, but not do this |
| `not_found` | 404 `not_found` | the actor must not learn that it exists |
| `reauthenticate` | 403 `reauthentication_required` | the actor may, after proving again who they are |

The difference between `forbid` and `not_found` matters. A member of the invoice's organization may see it, so refusing to send it is a 403. Someone from another organization learns nothing, not even that invoice 42 exists, so they get the same 404 as for a missing invoice. Invoice IDs reveal nothing.

**A policy decides from the actor, the database, and the path, never from the body.** That is what lets a page ask a policy before it renders a form, when only the form's path is known. A rule about the body, such as "the customer named in the request must be one of yours", is the operation's own, answered with its own result: `Invoices::Create` returns `UnknownCustomer` for another organization's customer ([Results, Errors, and Status Codes](results-errors-and-status-codes.md)). A rule about state, such as "a sent invoice cannot be sent again", is the operation's too.

The policy runs in the operation's transaction, after the actor is loaded, so a role revoked a moment ago is revoked for this request, and a Command's decision cannot be raced by a concurrent change. Keep a policy to one indexed lookup: a page with 50 links asks it 50 times.

## Grants: Commands cannot skip their policy

A Command with a policy must take `Punch::Grant(Self)` in `call`. Only the policy can make one. So code that calls the Command directly, a page, a job, or another Command, has to ask the policy first:

```crystal illustrative="calling a Command from code: ask its policy for the grant"
case grant = Punch::Grant(Send).check(actor, db, invoice_id: id)
in Punch::Grant(Send)     then Send.new.call(input, db, actor, grant)
in Punch::Policy::Refused then grant
end
```

`Punch::Grant(...).new` does not compile. A Command cannot be called around its policy by accident.

## Pages ask the same policy

On a page that takes the actor, a form for a Command and a link to a page ask the target's policy, with the path parameters they were built with, before they render. When the viewer may not, the element is left out, with everything inside it. The reference app's invoice page shows the Send button to an editor and not to a member, without a line of code that compares roles:

```crystal source=reference/spec/accounts/roles_spec.cr select="roles lets a member see invoices but not send them: 403, and no button"
# spec/accounts/roles_spec.cr
# ...

describe "roles" do
  it "lets a member see invoices but not send them: 403, and no button" do
    _, bob, organization, ada_client, bob_client = two_users
    id = start_draft(ada_client, add_customer(ada_client))
    AccountsSpec.give(bob.id, :member, on: organization)
    bob_client.post(Accounts::Organizations::Switch.path(organization_id: organization))

    bob_client.get(Billing::Invoices::Show.path(invoice_id: id)).status.should eq(HTTP::Status::OK)
    sent = bob_client.post(Billing::Invoices::Send.path(invoice_id: id), headers: Punch::Spec.idempotency_key)
    expect_forbidden(sent)
    sent.trace.root["punch.roles"].should eq("accounts.roles.member")
    expect_forbidden(bob_client.post(Billing::Customers::Create.path, json: {name: "Eve", email: "eve@example.org"}))

    page = bob_client.visit(Billing::Invoices::ShowPage.href(invoice_id: id))
    page.status.should eq(HTTP::Status::OK)
    page.should_not have_element("button", text: "Send invoice")
    page.should_not have_element("button", text: "Delete draft")
    bob_client.visit(Home::IndexPage.href).should_not have_element("a", text: "New invoice")
    bob_client.visit(Billing::Invoices::CreatePage.href).status.should eq(HTTP::Status::FORBIDDEN)
    bob_client.submit(Billing::Invoices::Send.action(invoice_id: id)).status.should eq(HTTP::Status::FORBIDDEN)

    AccountsSpec.give(bob.id, :editor, on: organization)
    bob_client.visit(Billing::Invoices::ShowPage.href(invoice_id: id)).should have_element("button", text: "Send invoice")
    bob_client.post(Billing::Invoices::Send.path(invoice_id: id), headers: Punch::Spec.idempotency_key).status.should eq(HTTP::Status::OK)
  end

  # ...
end
```

So never hide a button with an `if` of your own. The rule would then live in two places, and the day they disagree, the page offers a button that answers 403, or hides one that would work. The target still refuses on its own, so a hidden button is a convenience, never the protection.

## Lists are authorized in SQL

A policy decides about one call, not about rows. A list selects only what the viewer may see, in its query, so pages and counts stay right. The reference app's invoice list filters by the organization the actor works in. With roles, a query can join Punch's view `punch_permissions`, which holds who may run what, on which resource:

```sql source=reference/app/accounts/organizations/queries.sql select=Mine
-- app/accounts/organizations/queries.sql
-- name: Mine :many
-- The organizations a user holds a role on, by name.
-- sort: the organizations of one user, a few rows
-- param user_id: Int64
SELECT organizations.* FROM organizations
WHERE organizations.id IN (
  SELECT p.resource_id FROM punch_permissions p
  WHERE p.user_id = :user_id AND p.resource_type = 'accounts.organizations')
ORDER BY organizations.name, organizations.id;
```

## Roles

A role is a set of operations, defined in code. Who holds it, on which resource, and until when is data. The reference app's roles:

```crystal source=reference/app/accounts/roles.cr select=Accounts::Roles::Operator,Accounts::Roles::Member,Accounts::Roles::Editor,Accounts::Roles::Administrator
# app/accounts/roles.cr
module Accounts::Roles
  # Runs the app: the ops console at /_punch, in every organization and
  # outside them. Given on the box with `bin/invoicing roles:assign
  # Accounts::Roles::Operator --email ADDRESS`.
  struct Operator
    include Punch::Role

    permits Punch::Ops::Console::Operations
  end

  # Sees an organization's customers and invoices, and who belongs to it.
  struct Member
    include Punch::Role

    scope Organizations::ID
    permits Organizations::Switch, Organizations::Members::Index, Organizations::Members::IndexPage,
      Billing::Customers::Index, Billing::Customers::IndexPage, Billing::Customers::Search,
      Billing::Customers::Show, Billing::Customers::UpdatePage, Billing::Customers::Statement,
      Billing::Invoices::Index, Billing::Invoices::IndexPage, Billing::Invoices::Show, Billing::Invoices::ShowPage
  end

  # A member who also adds and changes customers, writes invoices, sends
  # them, and shares their links.
  struct Editor
    include Punch::Role

    scope Organizations::ID
    permits Member,
      Billing::Customers::Create, Billing::Customers::CreatePage, Billing::Customers::Update,
      Billing::Invoices::Create, Billing::Invoices::CreatePage, Billing::Invoices::Update,
      Billing::Invoices::Destroy, Billing::Invoices::Send, Billing::Invoices::Link
  end

  # An editor who also decides who belongs to the organization, in
  # which role.
  struct Administrator
    include Punch::Role

    scope Organizations::ID
    permits Editor, Organizations::Members::Create, Organizations::Members::Destroy
  end

  # ...
end
```

- `scope` names the resource a role is held on, by its typed ID: here, an organization. A role without a scope is global, held everywhere, like `Operator`, who runs the ops console.
- `permits` lists operations, and other roles of the same scope whose operations it includes. Pages are operations too: list the page beside the Query it shows.
- Permissions are operations, named by their IDs. There are no permission strings to keep in sync with the code, and a role that permits nothing does not compile.

A policy asks the roles with `Punch::Roles.check(db, user_id, operation, on: resource)`, which answers `allow`, `forbid` when the user holds a role there that does not permit this, or `not_found` when they hold nothing there. It reads the user's assignments once per transaction, so a page of 50 links costs one query for the roles.

**Giving and taking roles.** `Punch::Roles.assign` and `Punch::Roles.revoke` change assignments inside a Command, which publishes `Punch::Roles::Assigned` or `Revoked`. The reference app's `Accounts::Organizations::Members` Commands do this, and keep at least one administrator per organization. On the box, `punch roles:assign Accounts::Roles::Operator --email ADDRESS` gives a global role, such as the one that opens the ops console. `punch roles` lists every role with its operations, and `punch roles --user ID` a user's assignments ([Authorization: commands](../authorization.md#commands)).

**Working in one organization.** A user can belong to several organizations. The reference app's `Accounts::Organizations::Current` policy decides operations on the organization the user chose to work in, such as listing invoices or creating one ([Authorization: working in one organization](../authorization.md#working-in-one-organization)).

## Recent authentication

Changing how someone signs in, deleting an account, or moving money should not be possible from a browser someone left open. A policy that includes `Punch::Policy::RecentAuthentication` answers `reauthenticate` unless the session proved who the user is within the last 15 minutes. A browser then goes to the app's re-authentication page and back ([Authorization: recent authentication](../authorization.md#recent-authentication)).

## Testing

Spec a policy through its operations, as the roles spec above does: one client per user, the role given, the status and the page checked. `Punch::Spec.grant(Command, actor, writer, …)` makes a grant for a spec that calls a Command directly. Every refused request's trace records the policy and the roles that decided, `punch.policy` and `punch.roles`.

When a user may not do something they expected to, `punch authz:explain OPERATION --user ID invoice_id=42` runs the policy as a request would and says why: the decision, the roles that decided, which roles would permit it, and what the user holds.

## Common mistakes

**A rule about the body in a policy.** A policy cannot see the body, by design. Check it in the operation and return a result.

**Answering 403 to someone from another organization.** It confirms the resource exists. Return `not_found` to anyone who holds nothing there.

**Hiding a button with `if actor.admin?`.** The rule is then in two places. Render the form, and let its policy leave it out.

**Filtering a list in Crystal.** Reading every row and dropping what the viewer may not see breaks page sizes and reads too much. Filter in the query.

**`@[Punch::Public]` to make a compile error go away.** Public means anyone on the internet. Write the policy.

## Limits

- A policy sees the path, not the body.
- Roles are permissions on operations, not on fields or rows; a rule about which rows is the list query's.
- There are no role hierarchies across scopes: a role includes only roles of its own scope.

## Rules

- Give every operation `@[Punch::Authorize(Policy)]` or `@[Punch::Public]`; public only for what anyone may do. ([Every operation says who may run it](#every-operation-says-who-may-run-it))
- Decide in a policy from the actor, the database, and the path; check the body and state in the operation. ([Policies](#policies))
- Answer `not_found` where the actor must not learn that something exists. ([Policies](#policies))
- Keep a policy to one indexed lookup. ([Policies](#policies))
- Call a Command from code only with a grant from `Punch::Grant(...).check`. ([Grants: Commands cannot skip their policy](#grants-commands-cannot-skip-their-policy))
- Never hide forms or links by hand; render them, and let their policies decide. ([Pages ask the same policy](#pages-ask-the-same-policy))
- Scope lists in SQL. ([Lists are authorized in SQL](#lists-are-authorized-in-sql))

## Reference

- [Authorization](../authorization.md): declaring, policies, when they run, recent authentication, grants, forms and links, roles, assignments, tables, commands, and spec helpers.
- [Authentication](../auth.md): the actor and re-authentication.
