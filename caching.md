# Caching

Applies to Punch 0.1.

In a Punch app, the database is in the same process as the code, so reading a row costs about as much as reading a cached copy of it. Most apps that cache rows elsewhere need not here. What is still worth caching is what is expensive to produce: a heavy aggregate, a document rendered from many rows, the answer of another system. Punch has typed cache stores for those, forgotten by domain events when the data behind them changes, and checks that the app works just as well without them.

After reading this guide, you will know:

- what to cache in a SQLite app, and what not to;
- how to declare a cache store and fetch from it;
- how to keep a cached value current;
- why the specs of every app run twice;
- how to save clients a download with conditional requests.

Before you start: you know Queries and after-commit subscribers ([Operations and Routing](operations-and-routing.md), [Domain Events](domain-events.md)).

## What to cache

Do not cache rows. A read of `primary.db` is a lookup in the same process, and a cached copy is just a second place to keep correct.

Cache values that cost real work: an aggregate over many rows, a rendered document, the result of a call to another system. The reference app caches each customer's statement, which sums every sent invoice, per currency.

## A store and a fetch

A store is a constant, one per kind of value, with a name that is unique in the app. The statement's store and the Query that uses it:

```crystal source=reference/app/billing/customers/operations/statement.cr select=Billing::Customers::STATEMENTS,Billing::Customers::Statement
# app/billing/customers/operations/statement.cr
module Billing::Customers
  # ...

  # Statements for a minute, in memory in front of cache.db. Sending an
  # invoice or its payment forgets its customer's entry (`ForgetStatement`,
  # on `BalanceChanged`).
  STATEMENTS = Punch::Cache(ID, Balances).new("customer_statements", ttl: 1.minute, tier: :both)

  # What a customer owes, per currency.
  @[Punch::Authorize(ByRole)]
  struct Statement < Punch::Query
    struct Input
      include Punch::Input

      @[Punch::PathParam]
      getter customer_id : ID
    end

    alias Output = Balances | NotFound

    def call(input : Input, db : Punch::Reader, _actor : Invoicing::Actor, clock : Punch::Clock) : Output
      return NotFound.new unless Table.find(db, input.customer_id)

      STATEMENTS.fetch(input.customer_id) do
        now = clock.now
        Balances.new(input.customer_id, Queries.statement(db, now: now, customer_id: input.customer_id).map { |row| Balance.new(row) }, now)
      end
    end
  end
end
```

`Punch::Cache(ID, Balances)` says the keys are customer IDs and the values are `Balances`. Keys are simple values: strings, integers, enums, domain primitives, typed IDs, or tuples of those. Values must serialize to JSON. A value that holds a `Punch::Secret` anywhere does not compile, so a secret never lands in the cache file.

`fetch(key) { … }` returns the stored value, or runs the block, stores its result, and returns it. When several requests miss the same key at once, one computes and the others wait for its result.

Each store has a time-to-live and a tier:

| `tier:` | Kept in | |
|---|---|---|
| `:memory` (the default) | the process | fastest; lost on restart; each process has its own |
| `:durable` | `cache.db` | survives restarts; shared by the processes on the machine |
| `:both` | memory, in front of `cache.db` | the statement's choice |

Fetch in a Query, not in a Command. Inside a Command's transaction, the block would see uncommitted data, and a rollback would leave a value that was never true.

## Keeping a value current

The time-to-live bounds how old a value can get. To forget it as soon as the data behind it changes, name the store on an after-commit subscriber with `@[Punch::Subscriber::Forget]`, and Punch writes the subscriber's `call`:

```crystal source=reference/app/billing/customers/subscribers/forget_statement.cr
# app/billing/customers/subscribers/forget_statement.cr
module Billing::Customers
  # A sent or paid invoice changes what its customer owes, so the cached
  # statement is forgotten once the change is committed; the next request
  # computes it again. Without this, the statement would be up to a minute
  # old. Punch writes the `call` (`punch events` shows the store).
  @[Punch::Subscriber::Forget(STATEMENTS, key: customer_id)]
  struct ForgetStatement
    include Punch::Subscriber(BalanceChanged)
  end
end
```

`key:` names the event's getter that returns the key, here `customer_id`. The subscriber listens to `BalanceChanged`, which every event that changes what a customer owes includes, so a new such event cannot forget to forget. Entries are forgotten after the commit, so a Command that fails forgets nothing, and a reader cannot recompute a value from data about to roll back. `punch events` shows which store each subscriber forgets.

The reference app's spec checks it:

```crystal source=reference/spec/billing/customers/forget_statement_spec.cr select="customer statement forgets the statement when an invoice to the customer is sent"
# spec/billing/customers/forget_statement_spec.cr
# ...

describe "customer statement" do
  it "forgets the statement when an invoice to the customer is sent" do
    client = AccountsSpec.client
    customer = customer_id(client)
    send_invoice(client, customer, 1_000)
    path = Billing::Customers::Statement.path(customer_id: customer)
    first = client.get(path)
    etag = first.headers["ETag"]

    -> { send_invoice(client, customer, 2_000) }.should publish(Billing::InvoiceSent, customer_id: customer, times: 1)
    Billing::Customers::STATEMENTS.cached?(customer).should be_false
    now = client.get(path, headers: HTTP::Headers{"If-None-Match" => etag})
    now.status.should eq(HTTP::Status::OK)
    balances(now).should eq([{"EUR", 2, 0, 3_000}])
  end
end
```

**Other processes.** Forgetting reaches `cache.db` and the memory of the process that committed the change. Another process's memory keeps its copy until it expires: a separate jobs process, or the old process during a deploy. Choose `:durable` for a store whose values must not outlive such a change, or keep its time-to-live short.

## Correct with an empty cache

The cache is empty after a restart, a restore, or `punch cache:clear`. So an app must give the same answers without it, only slower. `punch check` proves this: after the specs pass, it runs them again with `PUNCH_CACHE=off`, where every fetch computes (its `nocache` step). That is why every app's specs run twice.

A spec that checks caching itself branches on `Punch::Caches.enabled?`. `Punch::Spec.cache_clock` lets entries expire in a spec, and `STATEMENTS.cached?(key)` checks that a change forgot one ([Cache: specs](../cache.md#specs)).

## Conditional requests

A browser or client that asks for the same statement again could be told "unchanged" instead of getting the body again. A Query result that includes `Punch::HTTP::Versioned` gets an `ETag`, and a request with that tag in `If-None-Match` gets `304 Not Modified` without a body. The statement's result is one:

```crystal source=reference/app/billing/customers/operations/statement.cr select=Billing::Customers::Balances
# app/billing/customers/operations/statement.cr
module Billing::Customers
  # ...

  # A customer's balances per currency. Computing it adds up every line of
  # every sent invoice, so it is cached (`STATEMENTS`) and may be up to a
  # minute old; `as_of` says when it was computed. Repeated requests get
  # `304 Not Modified` while the balances are unchanged.
  @[Punch::Status(200)]
  struct Balances
    include JSON::Serializable
    include Punch::HTTP::Versioned

    getter customer_id : ID
    getter balances : Array(Balance)
    getter as_of : Time

    def initialize(@customer_id : ID, @balances : Array(Balance), @as_of : Time)
    end

    # The balances themselves: a recomputed statement with the same
    # numbers keeps its version.
    def version : String
      Digest::SHA256.hexdigest(balances.to_json)
    end
  end

  # ...
end
```

Its `version` is a digest of the balances alone, so a statement recomputed with the same numbers keeps its tag, while `as_of` changes.

The tag covers the result type, the signed-in user, the version, and the running binary, so users never share a tag and a deploy changes every tag. A 304 saves the transfer, not the work: the result is computed before the tag is compared. Cache the result too when computing it is what costs, as the statement does.

## Inspecting

The ops console's cache page, `/_punch/cache`, shows each store's hits, misses, and size in the serving process, and the cache file's size against its cap. `punch cache:stats` lists entries per store in `cache.db`, and `punch cache:clear [STORE]` removes them. Every fetch counts as a hit or miss on the request's wide event ([Tracing](../tracing.md)).

`cache.db` is capped at 256 MiB (`PUNCH_CACHE_MB`); beyond that, entries closest to expiry go first. It is not backed up, since everything in it can be computed again.

## Common mistakes

**Caching rows.** It adds a second copy to keep correct and saves almost nothing. Read the row.

**Fetching in a Command.** The value is computed from uncommitted data. Fetch in a Query, and forget from an after-commit subscriber.

**Forgetting in the Command.** A `STORE.delete(key)` in `call` happens before the commit; a reader can compute the old value again in between. Use `@[Punch::Subscriber::Forget]`.

**A spec that passes only with the cache.** `punch check` fails on `nocache`. The app must not depend on the cache for correctness.

**Expecting a change to reach every process at once.** Memory tiers of other processes keep their copy until it expires. Use `:durable` or a short time-to-live where that matters.

## Limits

- Forgetting does not reach the memory of other processes.
- A 304 saves the response, not the computation.
- `cache.db` is per machine and not backed up.

## Rules

- Cache what is expensive to produce, never rows. ([What to cache](#what-to-cache))
- Declare one store per kind of value, as a constant with a unique name. ([A store and a fetch](#a-store-and-a-fetch))
- Fetch in a Query; forget with `@[Punch::Subscriber::Forget]` after commit. ([Keeping a value current](#keeping-a-value-current))
- Keep the app correct with an empty cache; `punch check` runs the specs with `PUNCH_CACHE=off`. ([Correct with an empty cache](#correct-with-an-empty-cache))

## Reference

- [Cache](../cache.md): keys, values, tiers, expiry, size, observability, conditional requests, and spec helpers.
- [Domain events](../events.md): after-commit subscribers.
